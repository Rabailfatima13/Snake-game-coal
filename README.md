; ==========================================================================
; SNAKE GAME - 8088 ASSEMBLY IMPLEMENTATION (NASM VERSION)
; 
; A classic Snake game implementation for 8088 processors
; Running in text mode (80x25)
; ==========================================================================

[org 100h]       ; COM program starts at offset 100h

section .text
    jmp start

; ==========================================================================
; CONSTANTS & VARIABLES
; ==========================================================================
    ; Game field boundaries
    FIELD_WIDTH  equ 40      ; Game field width
    FIELD_HEIGHT equ 20      ; Game field height
    
    ; Game states
    STATE_PLAYING   equ 0
    STATE_GAME_OVER equ 1
    
    ; Direction constants
    DIR_RIGHT   equ 0
    DIR_DOWN    equ 1
    DIR_LEFT    equ 2
    DIR_UP      equ 3
    
    ; Characters to use
    SNAKE_HEAD_CHAR   equ 2  ; ASCII character for snake head (smiley face)
    SNAKE_BODY_CHAR   equ 'o'; ASCII character for snake body
    FOOD_CHAR         equ '*'; ASCII character for food
    WALL_CHAR         equ 219; ASCII character for wall (full block)
    EMPTY_CHAR        equ ' '; Empty space character
    
    ; Colors
    BLACK       equ 0
    BLUE        equ 1
    GREEN       equ 2
    CYAN        equ 3
    RED         equ 4
    MAGENTA     equ 5
    BROWN       equ 6
    LIGHT_GRAY  equ 7
    DARK_GRAY   equ 8
    LIGHT_BLUE  equ 9
    LIGHT_GREEN equ 10
    LIGHT_CYAN  equ 11
    LIGHT_RED   equ 12
    LIGHT_MAGENTA equ 13
    YELLOW      equ 14
    WHITE       equ 15

; ==========================================================================
; CODE SECTION
; ==========================================================================
start:
    ; Set video mode (text mode 80x25, color)
    mov ah, 00h
    mov al, 03h
    int 10h
    
    ; Hide cursor
    mov ah, 01h
    mov cx, 2000h  ; Set cursor out of visible range
    int 10h
    
START_GAME:
    call ClearScreen
    call InitGame
    call DrawBorder
    call GenerateFood
    
GAME_LOOP:
    ; Check if any key is pressed
    mov ah, 01h
    int 16h
    jz NO_KEY_PRESSED
    
    ; Get the key
    mov ah, 00h
    int 16h
    
    ; Check for arrow keys (scan codes)
    cmp ah, 48h     ; Up arrow
    je UP_PRESSED
    cmp ah, 50h     ; Down arrow
    je DOWN_PRESSED
    cmp ah, 4Bh     ; Left arrow
    je LEFT_PRESSED
    cmp ah, 4Dh     ; Right arrow
    je RIGHT_PRESSED
    
    ; Check for other control keys
    cmp al, 27      ; ESC key
    je EXIT_GAME
    cmp al, 'r'     ; R key (restart)
    je START_GAME
    cmp al, 'R'     ; R key (uppercase)
    je START_GAME
    
    jmp NO_KEY_PRESSED
    
UP_PRESSED:
    mov al, [snakeDir]
    cmp al, DIR_DOWN   ; Cannot move up if currently moving down
    je NO_KEY_PRESSED
    mov byte [snakeDir], DIR_UP
    jmp NO_KEY_PRESSED
    
DOWN_PRESSED:
    mov al, [snakeDir]
    cmp al, DIR_UP     ; Cannot move down if currently moving up
    je NO_KEY_PRESSED
    mov byte [snakeDir], DIR_DOWN
    jmp NO_KEY_PRESSED
    
LEFT_PRESSED:
    mov al, [snakeDir]
    cmp al, DIR_RIGHT  ; Cannot move left if currently moving right
    je NO_KEY_PRESSED
    mov byte [snakeDir], DIR_LEFT
    jmp NO_KEY_PRESSED
    
RIGHT_PRESSED:
    mov al, [snakeDir]
    cmp al, DIR_LEFT   ; Cannot move right if currently moving left
    je NO_KEY_PRESSED
    mov byte [snakeDir], DIR_RIGHT
    
NO_KEY_PRESSED:
    ; Game logic
    call UpdateSnake
    call DrawScore
    
    ; Check game state
    mov al, [gameState]
    cmp al, STATE_GAME_OVER
    je HANDLE_GAME_OVER
    
    ; Delay for game speed
    call Delay
    
    ; Continue game loop
    jmp GAME_LOOP
    
HANDLE_GAME_OVER:
    ; Show Game Over message
    mov ah, 02h
    mov bh, 0       ; Page number
    mov dh, FIELD_HEIGHT + 2  ; Row
    mov dl, 10      ; Column
    int 10h
    
    mov ah, 09h
    mov al, ' '     ; Character
    mov bl, WHITE   ; Attribute
    mov cx, 50      ; Number of times to write
    int 10h
    
    mov ah, 02h
    mov bh, 0       ; Page number
    mov dh, FIELD_HEIGHT + 2  ; Row
    mov dl, 10      ; Column
    int 10h
    
    ; Print Game Over message
    mov si, gameOverMsg
    call PrintString
    
    ; Wait for key press
    WAIT_KEY:
        mov ah, 00h
        int 16h
        
        cmp al, 27      ; ESC key
        je EXIT_GAME
        cmp al, 'r'     ; R key (restart)
        je START_GAME
        cmp al, 'R'     ; R key (uppercase)
        je START_GAME
        
        jmp WAIT_KEY
    
EXIT_GAME:
    ; Restore cursor
    mov ah, 01h
    mov cx, 0607h
    int 10h
    
    ; Exit to DOS
    mov ah, 4Ch
    int 21h

; ==========================================================================
; PROCEDURE: InitGame
; Initialize game state and snake position
; ==========================================================================
InitGame:
    ; Set initial game state
    mov byte [gameState], STATE_PLAYING
    mov byte [snakeDir], DIR_RIGHT
    mov word [snakeLength], 3
    mov word [score], 0
    
    ; Initialize snake position (middle of screen)
    mov cx, 3              ; Initial length
    mov si, 0              ; Array index
    mov bl, 10             ; Initial X position
    mov bh, 10             ; Initial Y position
    
INIT_SNAKE_LOOP:
    mov [snakePos + si], bx
    dec bl                 ; Move initial segments to the left
    add si, 2              ; Move to next array element
    loop INIT_SNAKE_LOOP
    
    ret

; ==========================================================================
; PROCEDURE: ClearScreen
; Clears the screen
; ==========================================================================
ClearScreen:
    mov ah, 06h    ; Scroll up function
    mov al, 0      ; Clear entire screen
    mov bh, 07h    ; Normal attribute (White on Black)
    mov cx, 0      ; Upper left corner (0,0)
    mov dh, 24     ; Lower right corner (row 24)
    mov dl, 79     ; Lower right corner (column 79)
    int 10h
    
    ; Print welcome message
    mov ah, 02h
    mov bh, 0      ; Page number
    mov dh, 0      ; Row
    mov dl, 10     ; Column
    int 10h
    
    mov si, welcomeMsg
    call PrintString
    
    ret

; ==========================================================================
; PROCEDURE: DrawBorder
; Draws the game border
; ==========================================================================
DrawBorder:
    ; Top border
    mov ah, 02h      ; Set cursor position
    mov bh, 0        ; Page number
    mov dh, 1        ; Row
    mov dl, 0        ; Column
    int 10h
    
    mov cx, FIELD_WIDTH + 2  ; Width of border
    mov al, WALL_CHAR        ; Character
    mov bl, YELLOW           ; Color attribute
    mov ah, 09h              ; Display character function
    
TOP_BORDER:
    int 10h
    inc dl           ; Move cursor
    mov ah, 02h
    int 10h
    mov ah, 09h
    loop TOP_BORDER
    
    ; Bottom border
    mov ah, 02h
    mov dh, FIELD_HEIGHT + 2  ; Row
    mov dl, 0                 ; Column
    int 10h
    
    mov cx, FIELD_WIDTH + 2   ; Width of border
    mov al, WALL_CHAR         ; Character
    mov bl, YELLOW            ; Color attribute
    mov ah, 09h
    
BOTTOM_BORDER:
    int 10h
    inc dl           ; Move cursor
    mov ah, 02h
    int 10h
    mov ah, 09h
    loop BOTTOM_BORDER
    
    ; Left and right borders
    mov cx, FIELD_HEIGHT    ; Height of play area
    mov dh, 2              ; Starting row
    
SIDE_BORDERS:
    ; Left border
    mov ah, 02h
    mov dl, 0             ; Left column
    int 10h
    
    mov ah, 09h
    mov al, WALL_CHAR     ; Character
    mov bl, YELLOW        ; Attribute
    int 10h
    
    ; Right border
    mov ah, 02h
    mov dl, FIELD_WIDTH + 1  ; Right column
    int 10h
    
    mov ah, 09h
    mov al, WALL_CHAR     ; Character
    mov bl, YELLOW        ; Attribute
    int 10h
    
    inc dh               ; Next row
    loop SIDE_BORDERS
    
    ret

; ==========================================================================
; PROCEDURE: UpdateSnake
; Updates snake position and checks collisions
; ==========================================================================
UpdateSnake:
    ; Get current head position
    mov si, 0                ; Index to first segment (head)
    mov bx, [snakePos + si]  ; BL = X, BH = Y
    mov dl, bl              ; Copy X to DL
    mov dh, bh              ; Copy Y to DH
    
    ; Calculate new head position based on direction
    mov al, [snakeDir]
    cmp al, DIR_RIGHT
    je MOVE_RIGHT
    cmp al, DIR_DOWN
    je MOVE_DOWN
    cmp al, DIR_LEFT
    je MOVE_LEFT
    cmp al, DIR_UP
    je MOVE_UP
    
MOVE_RIGHT:
    inc dl
    jmp CHECK_COLLISION
    
MOVE_DOWN:
    inc dh
    jmp CHECK_COLLISION
    
MOVE_LEFT:
    dec dl
    jmp CHECK_COLLISION
    
MOVE_UP:
    dec dh
    
CHECK_COLLISION:
    ; Check wall collision
    cmp dl, 0
    je COLLISION
    cmp dl, FIELD_WIDTH + 1
    je COLLISION
    cmp dh, 1
    je COLLISION
    cmp dh, FIELD_HEIGHT + 2
    je COLLISION
    
    ; Check self collision (by checking if new position matches any segment)
    mov si, 0
    mov cx, [snakeLength]
    
CHECK_SELF_LOOP:
    mov bx, [snakePos + si]  ; Get segment position
    cmp bl, dl            ; Compare X coordinates
    jne NEXT_SEGMENT
    cmp bh, dh            ; Compare Y coordinates
    je COLLISION
    
NEXT_SEGMENT:
    add si, 2             ; Next segment
    loop CHECK_SELF_LOOP
    
    ; Check food collision
    mov al, [foodX]
    cmp al, dl
    jne NO_FOOD_COLLISION
    mov al, [foodY]
    cmp al, dh
    jne NO_FOOD_COLLISION
    
    ; Food collision occurred, increase snake length
    inc word [snakeLength]
    add word [score], 10
    call GenerateFood
    
NO_FOOD_COLLISION:
    ; Move snake body segments
    mov cx, [snakeLength]
    mov si, cx
    shl si, 1             ; SI = snakeLength * 2
    sub si, 2             ; Index of the last segment
    
MOVE_SEGMENTS:
    mov bx, [snakePos + si - 2]
    mov [snakePos + si], bx
    sub si, 2
    cmp si, 0
    jg MOVE_SEGMENTS
    
    ; Set new head position
    mov bl, dl
    mov bh, dh
    mov [snakePos], bx
    
    ; Draw all snake segments
    call DrawSnake
    
    ret
    
COLLISION:
    mov byte [gameState], STATE_GAME_OVER
    ret

; ==========================================================================
; PROCEDURE: DrawSnake
; Draws the snake on the screen
; ==========================================================================
DrawSnake:
    ; First clear the screen area (not border)
    mov ah, 06h      ; Scroll up function
    mov al, 0        ; Clear
    mov bh, 0        ; Attribute
    mov cx, 0102h    ; Upper left (row 2, column 1)
    mov dx, 0142h    ; Lower right (based on field size)
    add dh, FIELD_HEIGHT
    add dl, FIELD_WIDTH
    int 10h
    
    ; Draw food first
    mov ah, 02h      ; Set cursor position
    mov bh, 0        ; Page number
    mov dl, [foodX]  ; Column
    mov dh, [foodY]  ; Row
    int 10h
    
    mov ah, 09h      ; Write character
    mov al, FOOD_CHAR
    mov bl, LIGHT_RED
    mov cx, 1        ; Just one character
    int 10h
    
    ; Draw snake segments
    mov si, 0        ; Start with the head
    mov cx, [snakeLength]
    
DRAW_SEGMENTS:
    mov bx, [snakePos + si]
    
    mov ah, 02h      ; Set cursor position
    mov dl, bl       ; X position
    mov dh, bh       ; Y position
    int 10h
    
    mov ah, 09h      ; Write character
    cmp si, 0        ; Check if this is the head
    jne DRAW_BODY
    mov al, SNAKE_HEAD_CHAR
    mov bl, LIGHT_GREEN
    jmp DRAW_CHAR
    
DRAW_BODY:
    mov al, SNAKE_BODY_CHAR
    mov bl, GREEN
    
DRAW_CHAR:
    mov cx, 1        ; Just one character
    int 10h
    
    add si, 2        ; Next segment
    loop DRAW_SEGMENTS
    
    ret

; ==========================================================================
; PROCEDURE: GenerateFood
; Generates food at a random position
; ==========================================================================
GenerateFood:
    ; Generate random position for food
    call Random
    
    ; Scale to field size (1 to FIELD_WIDTH)
    mov ax, dx
    xor dx, dx
    mov bx, FIELD_WIDTH
    div bx          ; AX / BX = AX remainder DX
    inc dl          ; Avoid border at position 0
    mov [foodX], dl
    
    call Random
    
    ; Scale to field height (2 to FIELD_HEIGHT+1)
    mov ax, dx
    xor dx, dx
    mov bx, FIELD_HEIGHT
    div bx          ; AX / BX = AX remainder DX
    add dl, 2       ; Avoid top border (at row 1)
    mov [foodY], dl
    
    ; Check if food is colliding with snake
    mov si, 0
    mov cx, [snakeLength]
    
CHECK_FOOD_LOOP:
    mov bx, [snakePos + si]
    mov al, bl        ; Snake X
    cmp al, [foodX]
    jne NEXT_FOOD_CHECK
    mov al, bh        ; Snake Y
    cmp al, [foodY]
    jne NEXT_FOOD_CHECK
    
    ; Collision found, regenerate food
    call GenerateFood
    ret
    
NEXT_FOOD_CHECK:
    add si, 2
    loop CHECK_FOOD_LOOP
    
    ret

; ==========================================================================
; PROCEDURE: Random
; Generate pseudo-random number in DX
; ==========================================================================
Random:
    ; Simple linear congruential generator
    mov ax, [randomSeed]
    mov bx, 8405h
    mul bx
    inc ax
    mov [randomSeed], ax
    mov dx, ax
    ret

; ==========================================================================
; PROCEDURE: DrawScore
; Displays the current score
; ==========================================================================
DrawScore:
    ; Set cursor position
    mov ah, 02h
    mov bh, 0         ; Page number
    mov dh, FIELD_HEIGHT + 3  ; Row (below game area)
    mov dl, 10        ; Column
    int 10h
    
    ; Print "Score: " text
    mov si, scoreMsg
    call PrintString
    
    ; Convert score to ASCII
    mov ax, [score]
    mov bx, 10
    mov cx, 0
    
CONVERT_LOOP:
    xor dx, dx
    div bx
    push dx
    inc cx
    test ax, ax
    jnz CONVERT_LOOP
    
PRINT_LOOP:
    pop dx
    add dl, '0'
    mov ah, 02h
    int 21h
    loop PRINT_LOOP
    
    ret

; ==========================================================================
; PROCEDURE: PrintString
; Prints a null-terminated string pointed to by SI
; ==========================================================================
PrintString:
PRINT_LOOP_STR:
    mov al, [si]
    or al, al
    jz PRINT_DONE
    
    mov ah, 0Eh      ; BIOS teletype function
    int 10h
    
    inc si
    jmp PRINT_LOOP_STR
    
PRINT_DONE:
    ret

; ==========================================================================
; PROCEDURE: Delay
; Simple delay routine for game speed
; ==========================================================================
Delay:
    push cx
    push dx
    
    mov cx, 3        ; Adjust these values for game speed
    mov dx, 0FFFFh   ; Lower values = faster game
    
DELAY_LOOP:
    dec dx
    jnz DELAY_LOOP
    dec cx
    jnz DELAY_LOOP
    
    pop dx
    pop cx
    ret

; ==========================================================================
; DATA SECTION
; ==========================================================================
section .data
    gameState   db STATE_PLAYING    ; Current game state
    snakeDir    db DIR_RIGHT        ; Current snake direction
    snakeLength dw 3                ; Current length of snake
    score       dw 0                ; Player's score
    
    ; Food position e
    foodX       db 15
    foodY       db 10
    
    ; Random seed
    randomSeed  dw 12345

    ; Messages
    welcomeMsg  db "SNAKE GAME - Use arrow keys to control the snake", 0
    scoreMsg    db "Score: ", 0
    gameOverMsg db "GAME OVER! Press R to restart or ESC to exit", 0
    
    ; Snake position array (X,Y coordinates)
    ; Format: array of word values, each word contains:
    ; - Lower byte: X coordinate
    ; - Upper byte: Y coordinate
    snakePos    times 400 db 0      ; Can store up to 200 snake segments (200 words = 400 bytes)
