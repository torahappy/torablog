---
title: "Wine黒魔術"
tags: ["Wine", "MacOS", "システムコール", "黒魔術"]
date: 2020-05-09T01:18:41+09:00
---

WineでMacのシステムコールを呼んでみる！

MacやLinuxでexeファイルが実行できるWineは"Emulator"ではなく"Compatibility Layer"。つまり、WindowsをMacの中で動かしているのではなく、exeから呼び出されるdllをMac用の物に置き換えることでexeを動かしている。

普通は、windowsアプリのシステムコール（ウィンドウ表示、ファイル書き込みなどの動作）はdllを仲介して呼び出されているため、dllをすげ替えればほとんどのexeは動く。

でも実は直接、dllを介さずにシステムコールを呼ぶこともできる。
そんなことをしたら一体どうなるのだろうか・・・？

ということで、Wine上にウィンドウ表示するとともに、Mac OSにシステムコールを直接叩きつけるという闇exeを作ってみた！

なんと、Wineでダイアログが表示されると同時に、ちゃんと標準出力にもメッセージが出てきた！

### Assembly Code (MASM)

```assembly
.386
.model flat, stdcall
option casemap:none
include libs\windows.inc
include libs\kernel32.inc
includelib libs\kernel32.lib
include libs\user32.inc
includelib libs\user32.lib

.data
MsgCaption      db "Waiwai",0
MsgBoxText      db "GayagayaGayagaya",0
Filename        db "/Users/aidatorajiro/test",0

.code
start:
	push    ebp
	mov ebp, esp
	push sizeof MsgBoxText
	push offset MsgBoxText
	push 1
	mov     eax, 4
	sub     esp, 4
	int     80h
	add esp, 16
	pop ebp
	invoke MessageBox, NULL, addr MsgBoxText, addr MsgCaption, MB_OK
	invoke ExitProcess, NULL
end start
```

![](/img/winekur/scr0.jpg)

これを応用して、色々作ってみる。

## `wine safari.exe`とするとSafariを起動する

### Assembly Code (NASM)

```assembly
  global _main

  extern  _MessageBoxA@16
  extern  _ExitProcess@4


  section .text

_main:
  push    0
  push    message
  push    message
  push    0
  call   _MessageBoxA@16
    
    push    ebp
    mov ebp, esp
    push 0
    push argv
    push app_path
    mov     eax, 59
    sub     esp, 4
    int     80h
    add esp, 16
    pop ebp

  push    0
  call    _ExitProcess@4

  hlt

message:
  db      'Hello, Black Magic World!', 0
message_end:

app_path:
  db      '/Applications/Safari.app/Contents/MacOS/Safari', 0
app_path_end:

argv:
  dd app_path
  dd 0
argv_end:
```

![](/img/winekur/scr1.png)

↓

![](/img/winekur/scr2.png)

## 数をかぞえあげるだけ。エントロピーもゲットできる。

左したがエントロピー（システムコールを呼ぶとコンピュータがくれるランダムな数字）。
右上のカウントをクリックするとカウントが上がっていく。

GUIを作るの、めちゃくちゃめんどくさい。。。

### Assembly Code (NASM)

```assembly
global _main

extern  _MessageBoxA@16
extern  _CreateWindowExA@48
extern  _ExitProcess@4
extern  _ShowWindow@8
extern  _UpdateWindow@4
extern  _GetLastError@0
extern  _GetMessageA@16
extern  _TranslateMessage@4
extern  _DispatchMessageA@4
extern  _RegisterClassExA@4
extern  _DefWindowProcA@16
extern  _LoadIconA@8
extern  _LoadCursorA@8
extern  _GetWindowLongA@8
extern  _CreateWindowExA@48
extern  _DestroyWindow@4

SYS_fork equ 2
SYS_execve equ 59
SYS_getentropy equ 500

section .data

winmsg:
  times 7 dd 0

divisor_table:
  dd 1000000000
  dd 100000000
  dd 10000000
  dd 1000000
  dd 100000
  dd 10000
  dd 1000
  dd 100
  dd 10
  dd 1
  dd 0

temp_str:
  times 11 db 0
  db 10

temp_args:
  .hwin:
    dd 0
  .mess:
    dd 0
  .wpar:
    dd 0
  .lpar:
    dd 0

count_int:
  dd 0

window_long:
  dd 0

handle_window:
  dd 0

handle_button:
  dd 0

win_instance:
  dd 0

window_class:
  db      'BLACKMAGICK', 0

window_name:
  db      'うぃんどう', 0

message:
  db      'Hello, Black Magic World!', 0

message_ari:
  db      'ありだよ', 0

message_dame:
  db      'なしだよ', 0

btn_class:
  db      'BUTTON', 0

static_class:
  db      'STATIC', 0

safari_text:
  db      'さふぁり', 0

safari_id equ 152

exit_text:
  db      'おわる', 0

exit_id equ 153

count_text:
  db      'かうんと'

count_text_num:
  db '0000000000'
  db 0

count_id equ 154

count_handle:
  dd 0

count_text_handle:
  dd 0

entropy_btn_text:
  db 'えんとろぴい', 0

entropy_num_int:
  dd 0

entropy_num_text:
  db '0000000000'
  db 0

entropy_num_handle:
  dd 0

entropy_id equ 155

app_path:
  db      '/Applications/Safari.app/Contents/MacOS/Safari', 0

app_argv:
  dd app_path
  dd 0

window_class_data:
  dd (12*4) ; size
  dd 3 ; style
  dd func_win_callback ; proc
  dd 0
  dd 0
.instance:
  dd 0 ; instance
.icon:
  dd 0 ; icon
.cursor:
  dd 0 ; cursor
  dd (0 + 1) ; background
  dd 0 ; menuname
  dd window_class ; classname
  dd 0 ; icon

section .text

_main:
  ; |                |
  ; | INITialization |
  ; |                |

  mov eax, [esp + 4]
  mov [win_instance], eax
  
  push    0
  push    message
  push    message
  push    0
  call   _MessageBoxA@16
  
  mov eax, [win_instance]
  mov [window_class_data.instance], eax
  
  push 32512; IDI_APPLICATION
  push 0
  call _LoadIconA@8
  mov [window_class_data.icon], eax
  
  push 32514; IDC_ARROW + 2
  push 0
  call _LoadCursorA@8
  mov [window_class_data.cursor], eax
  
  push    window_class_data
  call   _RegisterClassExA@4
  
  push    0 ; param
  mov eax, [win_instance]
  push    eax ; instance
  push    0 ; menu
  push    0 ; parent
  push    600 ; h
  push    1000 ; w
  push    0x80000000 ; y
  push    0x80000000 ; x
  push    0x10000000 ; style
  push    window_name
  push    window_class
  push    0 ; exstyle
  call   _CreateWindowExA@48
  
  mov [handle_window], eax
  call func_err_test

  ; |               |
  ; | infinite loop |
  ; |               |
  
infinite:
  xor eax, eax

  push 0
  push 0
  push dword [handle_window]
  push winmsg
  call _GetMessageA@16
  
  cmp eax, 0 ; terminate
  je infinite_exit
  
  cmp eax, -1 ; error
  je infinite_exit
  
  push winmsg
  call _TranslateMessage@4
  
  push winmsg
  call _DispatchMessageA@4
  
  jmp infinite
  
infinite_exit:

  push    0
  call    _ExitProcess@4
  hlt

; |                           |
; | Windows callback function |
; |                           |

func_win_callback: ; handle, message, param, lparam
  mov eax, [esp + 4]
  mov [temp_args.hwin], eax ; hwin
  
  mov eax, [esp + 8]
  mov [temp_args.mess], eax ; mess
  
  mov eax, [esp + 12]
  mov [temp_args.wpar], eax ; wpar
  
  mov eax, [esp + 16]
  mov [temp_args.lpar], eax ; lpar

  ; |        |
  ; | CREATE |
  ; |        |
  
  cmp dword [temp_args.mess], 1 ; if create
  je .cond_m1
  jmp .cond_m1_r

  .cond_m1:

    ; get window long
    push -6
    push dword [temp_args.hwin]
    call _GetWindowLongA@8
    
    mov [window_long], eax

    ; create safari buttton
    push 0
    push dword [window_long]
    push safari_id
    push dword [temp_args.hwin]
    push 30
    push 100
    push 10
    push 10
    push 0x50000000
    push safari_text
    push btn_class
    push 0
    
    call _CreateWindowExA@48
    
    ; create exit buttton
    push 0
    push dword [window_long]
    push exit_id
    push dword [temp_args.hwin]
    push 30
    push 100
    push 90
    push 80
    push 0x50000000
    push exit_text
    push btn_class
    push 0
    
    call _CreateWindowExA@48

    ; create entropy buttton
    push 0
    push dword [window_long]
    push entropy_id
    push dword [temp_args.hwin]
    push 30
    push 100
    push 400
    push 30
    push 0x50000000
    push entropy_btn_text
    push btn_class
    push 0
    
    call _CreateWindowExA@48

    ; create entropy text
    call func_regenerate_entropy

    ; create count text & button
    call func_redraw_count_btn
  
  .cond_m1_r:

  ; |         |
  ; | COMMAND |
  ; |         |

  cmp dword [temp_args.mess], 0x111 ; if command
  je .cond_m2
  jmp .cond_m2_r

  .cond_m2:

    ; exit button pressed
    cmp word [temp_args.wpar], exit_id ; if exit_id
    je .flag_exit
    jmp .flag_not_exit
    
    .flag_exit:

    push dword [temp_args.hwin]
    call _DestroyWindow@4
    mov eax, 0
    ret

    .flag_not_exit:

    ; safari button pressed
    cmp word [temp_args.wpar], safari_id ; if safari_id
    je .flag_safari
    jmp .flag_not_safari

    .flag_safari:

    push    ebp
    mov ebp, esp
    push 0
    push app_argv
    push app_path
    mov     eax, SYS_fork
    sub     esp, 4
    int     80h
    add esp, 16
    pop ebp
    
    cmp edx, 0
    je .parent
    jmp .child
    
    .child:

    push    ebp
    mov ebp, esp
    push 0
    push app_argv
    push app_path
    mov     eax, SYS_execve
    sub     esp, 4
    int     80h
    add esp, 16
    pop ebp
    
    .parent:

    .flag_not_safari:

    ; count button pressed
    cmp word [temp_args.wpar], count_id ; if count_id
    je .flag_count
    jmp .flag_not_count

    .flag_count:

    add dword [count_int], 1
    mov eax, [count_int]
    mov ebx, count_text_num
    call func_int_to_str

    call func_redraw_count_btn

    .flag_not_count:

    ; entropy button pressed
    cmp word [temp_args.wpar], entropy_id ; if count_id
    je .flag_entropy
    jmp .flag_not_entropy

    .flag_entropy:

    call func_regenerate_entropy

    .flag_not_entropy:
  
  .cond_m2_r:

  ; |         |
  ; | DEFAULT |
  ; |         |
  
    push dword [temp_args.lpar]
    push dword [temp_args.wpar]
    push dword [temp_args.mess]
    push dword [temp_args.hwin]
    
    call _DefWindowProcA@16
    ret

; |                |
; | misc functions |
; |                |

; redraw entropy text
func_regenerate_entropy:
  cmp dword [entropy_num_handle],0
  je .create
  jmp .del

  .del:

  push dword [entropy_num_handle]
  call _DestroyWindow@4

  .create:

  push    ebp
  mov ebp, esp
  push 4
  push entropy_num_int
  mov     eax, SYS_getentropy
  sub     esp, 4
  int     80h
  add esp, 12
  pop ebp

  mov eax, [entropy_num_int]
  mov ebx, entropy_num_text
  call func_int_to_str

  push 0
  push dword [window_long]
  push 0
  push dword [temp_args.hwin]
  push 30
  push 100
  push 450
  push 30
  push 0x50000000
  push entropy_num_text
  push static_class
  push 0

  call _CreateWindowExA@48

  mov [entropy_num_handle], eax

; redraw count button and text
func_redraw_count_btn:
  cmp dword [count_handle],0
  je .create
  jmp .del

  .del:

  push dword [count_handle]
  call _DestroyWindow@4
  push dword [count_text_handle]
  call _DestroyWindow@4

  .create:

  push 0
  push dword [window_long]
  push count_id
  push dword [temp_args.hwin]
  push 30
  push 200
  push 180
  push 480
  push 0x50000000
  push count_text
  push btn_class
  push 0

  call _CreateWindowExA@48

  mov [count_handle], eax

  push 0
  push dword [window_long]
  push 0
  push dword [temp_args.hwin]
  push 30
  push 200
  push 300
  push 300
  push 0x50000000
  push count_text_num
  push static_class
  push 0
  
  call _CreateWindowExA@48

  mov [count_text_handle], eax

  ret

; print current error via MessageBox
func_print_err:
  push eax
  push ebx
  
  call   _GetLastError@0
  
  mov ebx, temp_str
  call func_int_to_str
  
  push    0
  push    message_dame
  push    temp_str
  push    0
  call   _MessageBoxA@16

  pop ebx
  pop eax
  ret

; chack if handle exists and print debug info via MessageBox
func_err_test: ; eax = handle
  cmp eax, 0
  je cond_whnd_t
  jmp cond_whnd_f

  cond_whnd_t:
    call func_print_err
    jmp cond_whnd_r
    
  cond_whnd_f:
    push    0
    push    message_ari
    push    message_ari
    push    0
    call   _MessageBoxA@16
    
    jmp cond_whnd_r

  cond_whnd_r:
    ret

; int to str
func_int_to_str: ; eax = number, ebx = pointer to string
  push eax
  push ebx
  push ecx
  push edx
  mov ecx, ebx
  mov ebx,divisor_table

  .loop:
    xor edx,edx
    div dword [ebx]
    add eax,'0'
    mov byte [ecx], al
    mov eax,edx
    add ecx,1
    add ebx,4
    cmp dword [ebx],0
  jne .loop
  
  pop edx
  pop ecx
  pop ebx
  pop eax
  ret
```

![](/img/winekur/scr3.png)

## さいごに

MacOS 10.15のリリースに伴い、32-bitのexeはWineでは実行できなくなってしまいました。。
さようなら、Wine....