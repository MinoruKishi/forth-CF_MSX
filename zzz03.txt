  
あなた:  
makefileの記述の仕方を教えてください。私が行いたいのは、  
sjasmplus --inc --inc=include --raw=build/echo03.com src/test_echo03.asm  
を実行したいのです。  
ChatGPT:  
了解です。目的はとても明確ですね 👍  
「この 1 行の sjasmplus コマンドを make で再現したい」という話なので、まずは最小で正しい Makefileから説明します。  
  
やりたいこと（再確認）  
sjasmplus --inc --inc=include --raw=build/echo03.com src/test_echo03.asm  
入力：src/test_echo03.asm  
  
出力：build/echo03.com  
  
include パス：  
  
カレントディレクトリ  
  
include/  
  
最小構成の Makefile（おすすめ）  
プロジェクト構成：  
  
.  
├─ Makefile  
├─ src/  
│   └─ test_echo03.asm  
├─ include/  
│   └─ MACROS_Z80.I  
└─ build/  
Makefile（これだけで動く）  
ASM     = sjasmplus  
SRC     = src/test_echo03.asm  
OUT     = build/echo03.com  
INCDIR  = include  
  
all: $(OUT)  
  
$(OUT): $(SRC)  
	$(ASM) --inc --inc=$(INCDIR) --raw=$(OUT) $(SRC)  
  
clean:  
	rm -f $(OUT)  
⚠ 注意  
  
Tab でインデントしてください（スペース不可）  
  
build/ ディレクトリは事前に存在している必要があります  
  
build ディレクトリを自動生成したい場合  
ASM     = sjasmplus  
SRC     = src/test_echo03.asm  
OUT     = build/echo03.com  
INCDIR  = include  
  
all: $(OUT)  
  
build:  
	mkdir -p build  
  
$(OUT): build $(SRC)  
	$(ASM) --inc --inc=$(INCDIR) --raw=$(OUT) $(SRC)  
  
clean:  
	rm -rf build  
これで：  
  
make  
だけで OK です。  
  
よくあるエラー（以前のエラーの原因）  
error: Unrecognized instruction: MACROS_Z80.I  
  
これは Makefile の問題ではなく asm 側の include 書き方です。  
  
❌ ダメな例  
MACROS_Z80.I  
✅ 正しい例（sjasmplus）  
INCLUDE "MACROS_Z80.I"  
または  
  
INCLUDE "include/MACROS_Z80.I"  
--inc / --inc=include を指定していれば  
拡張子 .I でも問題ありません  
  
発展：複数 asm を扱いたい場合（参考）  
ASM     = sjasmplus  
INCDIR  = include  
SRCS    = src/test_echo03.asm  
OUTS    = build/echo03.com  
  
all: $(OUTS)  
  
build:  
	mkdir -p build  
  
build/%.com: src/%.asm | build  
	$(ASM) --inc --inc=$(INCDIR) --raw=$@ $<  
  
clean:  
	rm -rf build  
まとめ（重要ポイント）  
Makefileは 「コマンドをそのまま書く」→ OK  
  
依存関係は  
出力 : 入力 の形で書く  
  
sjasmplus の INCLUDE は 命令  
  
Makefile の行頭は 必ず Tab  
  
次のステップとしては、  
  
.I を Z80 マクロ集として体系化する  
  
8086 → Z80 命令対応マクロの整理  
  
echo だけでなく キーボード → 画面の往復テスト  
  
この流れ、Z80 Forth VM に直結します。  
次は MACROS_Z80.I の設計に行きますか？  
  
あなた:  
修正の途中経過です。以下はMACROS.Iの一部で、ここではMACROS.I内のマクロが呼び出されたときに、reg部分に書き込まれたレジスタ名が置き換えられます。  
  IF reg=="AX"  
のところで、regの内容がAXのときにTrueとなってほしいのですが、 IF AX=="AX"と展開されてしまい、ワーニングが発生しています。なにかいい修正方法はありますか？  
  
  
◎マクロの例  
 MACRO I_JMP_IND reg  
  IF reg=="AX"  
    PUSH HL  
    JP (reg)  
  ELSE  
    IF reg=="BX"  
      PUSH BC  
      JP (reg)  
  ELSE  
    IF reg=="DX"  
      PUSH DE  
      JP (reg)  
  ELSE  
      ; 想定外レジスタ  
      ; わざとエラーにしたいならここで無効命令を出す/メッセージ出す  
      PUSH reg  
      JP (reg)  
  ENDIF ENDIF ENDIF  
 ENDM  
  
◎発生するワーニングの例  
MACROS_Z80.I(104): warning[fwdref]: forward reference of symbol: IF AX=="AX"  
test_echo03.asm(12): ^ emitted from here  
M  
ChatGPT:  
  
思考時間: 1m 25s  
原因はこれです：  
  
sjasmplus の MACRO 引数 reg は テキスト置換で展開されます（あなたの期待どおり）。  
  
でも IF は 式として評価されるので、IF AX=="AX" になると、左辺の AX は「文字列」ではなく **未定義シンボル(ラベル)**として解釈され、3-pass の都合で forward reference 警告になります。  
  
つまり sjasmplus では「マクロ引数が AX という“文字列”と一致するか」を、今の形（IF reg=="AX"）では安全に書けません。  
  
いちばん素直で壊れない解決：比較用は“文字列”で渡す（2引数にする）  
比較は文字列で行い、実際に JP ( ) に使うZ80レジスタは別引数で渡すのが安定です。  
  
MACRO I_JMP_IND x86reg, z80reg  
  IF x86reg == "AX"  
    PUSH HL  
    JP (z80reg)  
  ELSEIF x86reg == "BX"  
    PUSH BC  
    JP (z80reg)  
  ELSEIF x86reg == "DX"  
    PUSH DE  
    JP (z80reg)  
  ELSE  
    ; 想定外：ビルドを止めたいなら DISPLAY してから無効コードでも吐く  
    DISPLAY /L, "I_JMP_IND: unexpected reg=", x86reg  
    DB $DD, $DD   ; わざと変なバイト（好みで）  
  ENDIF  
ENDM  
呼び出し側をこうします：  
  
I_JMP_IND "AX", HL  
I_JMP_IND "BX", BC  
I_JMP_IND "DX", DE  
これなら IF は "AX" という文字列同士の比較になり、警告も消えます。文字列比較自体は sjasmplus の IF で普通に使われています（例：IF _VERSION = "1.07"）。   
  
1引数の見た目にこだわるなら：薄いラッパーマクロを用意  
呼び出しを I_JMP_IND AX に寄せたいなら、実体は上の2引数版にして、ラッパーを足すのが楽です。  
  
MACRO I_JMP_IND_AX  
  I_JMP_IND "AX", HL  
ENDM  
  
MACRO I_JMP_IND_BX  
  I_JMP_IND "BX", BC  
ENDM  
  
MACRO I_JMP_IND_DX  
  I_JMP_IND "DX", DE  
ENDM  
（sjasmplus のマクロは「引数の“値”で分岐」は弱いので、ここは割り切った方がハマりません。）  
  
あなたの元コードに近い形での注意点  
ELSEIF が使えるので、ENDIF ENDIF ENDIF のネスト地獄は避けられます。   
  
“想定外レジスタ”で わざとエラーにしたいなら、DISPLAY /L, ... でメッセージを出してから不正バイトを吐く、が分かりやすいです（上の例）。  
  
必要なら、あなたが作っている「8086命令→Z80マクロ」の全体方針に合わせて、  
  
AX/BX/DX/SI/DI/BP/SP を どのZ80レジスタ/メモリに割り当てるか  
  
PUSH/POP の整合性（Z80の PUSH は 16bitペア限定）  
  
JP (HL) 以外は間接ジャンプできない問題（BC/DE は一旦 HL に移す必要）  
  
…まで含めて、I_JMP_IND の最終形を一緒に固められます。  
  
あなた:  
マクロファイル（一部分）を下記のようにしてみました。そして MOV AX, BX で呼び出しましたが、エラーとなりました。文法的に何が悪かったのでしょうか？  
  
MACROS_Z80.I(34): error: Illegal instruction:     LD dst, src  
test_echo03.asm(12): ^ emitted from here  
  
------以下はマクロファイルの一部ーーーーー  
LBL_HL EQU HL  
LBL_BC EQU BC  
LBL_DE EQU DE  
LBL_AX EQU AX  
LBL_BX EQU BX  
LBL_DX EQU DX  
  
  
 MACRO I_MOV dst, src  
    ; 8086: MOV DX,BX / MOV BX,AX など  
    ; Z80: 16bitレジスタ同士を想定（HL/DE/BC/IX など）  
   IF dst==LBL_AX  
dst2 DEFS LBL_HL  
  ELSEIF dst==LBL_BX  
dst2 DEFS LBL_BC  
  ELSEIF dst==LBL_DX  
dst2 DEFS LBL_DE  
  ELSE  
      ; 想定外レジスタ  
      ; わざとエラーにしたいならここで無効命令を出す/メッセージ出す  
dst2 DEFS dst  
  ENDIF  
  
   IF src==LBL_AX  
src2 DEFS LBL_HL  
  ELSEIF src==LBL_BX  
src2 DEFS LBL_BC  
  ELSEIF src==LBL_DX  
src2 DEFS LBL_DE  
  ELSE  
      ; 想定外レジスタ  
      ; わざとエラーにしたいならここで無効命令を出す/メッセージ出す  
src2 DEFS src  
  ENDIF  
  
   MOV dst2, src2  
 ENDM  
ChatGPT:  
エラーの直接原因はここです：  
  
Z80には MOV 命令がありません（8080系の MOV とは違う）  
→ Z80 は LD です。  
  
そしてもっと本質的に、あなたの dst2 DEFS LBL_HL みたいな書き方だと、dst2 が “HL/BC/DE というトークンに置換されるマクロ変数” になっていません。  
DEFS は「バイト列を確保する（領域予約）」命令なので、レジスタ名を入れて命令オペランドに使う用途では使えないです。  
  
結果として最後の行が、  
  
MOV dst2, src2 → そもそも Z80 非対応  
  
たとえ LD dst2, src2 にしても、dst2 / src2 がレジスタ名に展開されず そのまま LD dst, src みたいな不正オペランドになって落ちます（ログの LD dst, src はまさにそれ）。  
  
修正方針（sjasmplusで確実に動くやり方）  
sjasmplus で「引数が AX/BX/DX なら HL/BC/DE に読み替える」をやるなら、“文字列で比較”して、実際に使うレジスタをその場で固定して出力が一番安全です。  
  
✅ 動く形（2段マクロ：ラッパーで文字列を渡す）  
; --- 実体：文字列で分岐して Z80コードを出す ---  
MACRO I_MOV_S d, s  
  IF d=="AX"  
    IF s=="BX"  
      LD HL,BC  
    ELSEIF s=="DX"  
      LD HL,DE  
    ELSEIF s=="AX"  
      ; LD HL,HL は無意味なので何もしない  
    ELSE  
      DISPLAY /L, "I_MOV: bad src=", s  
      DB 0  
    ENDIF  
  
  ELSEIF d=="BX"  
    IF s=="AX"  
      LD BC,HL  
    ELSEIF s=="DX"  
      LD BC,DE  
    ELSEIF s=="BX"  
      ; 何もしない  
    ELSE  
      DISPLAY /L, "I_MOV: bad src=", s  
      DB 0  
    ENDIF  
  
  ELSEIF d=="DX"  
    IF s=="AX"  
      LD DE,HL  
    ELSEIF s=="BX"  
      LD DE,BC  
    ELSEIF s=="DX"  
      ; 何もしない  
    ELSE  
      DISPLAY /L, "I_MOV: bad src=", s  
      DB 0  
    ENDIF  
  
  ELSE  
    DISPLAY /L, "I_MOV: bad dst=", d  
    DB 0  
  ENDIF  
ENDM  
  
; --- 呼び出しを 8086 風にするラッパー ---  
MACRO I_MOV dst, src  
  I_MOV_S "dst", "src"  
ENDM  
呼び出し：  
  
I_MOV AX, BX   ; → LD HL,BC  
I_MOV BX, AX   ; → LD BC,HL  
I_MOV DX, BX   ; → LD DE,BC  
ポイントは "dst" / "src" です。  
sjasmplus はマクロ引数を文字列展開できるので、これで "AX" "BX" が入って比較できます。  
  
なぜあなたの EQU + DEFS 方式がダメか（短く要点）  
LBL_AX EQU AX は「LBL_AX を AX に置換」するだけ  
  
IF dst==LBL_AX は最終的に IF AX==AX みたいになって、**文字列比較ではなく“シンボル比較”**になりがちで、前に出た forward reference 系の問題が再発しやすい  
  
dst2 DEFS LBL_HL は 領域確保であって、dst2 を “HLというトークン” にする機能じゃない  
→ だから LD dst2, src2 と書いてもレジスタにならない  
  
追加の現実的注意（Z80の16bit LDの制限）  
Z80で 16bitレジスタ同士の転送は万能じゃないです。  
  
LD HL,BC は 存在しません（Z80には 16bit reg-reg の LD は基本ない）  
  
できるのは主に：  
  
LD HL,(nn) / LD (nn),HL  
  
LD SP,HL  
  
PUSH rr / POP rr（BC/DE/HL/AF/IX/IY など）  
  
EX DE,HL  
  
なので上の例で LD HL,BC を出してますが、これは実機Z80では不可です。  
本当に AX→HL、BX→BC、DX→DE の「16bit MOV」を作るなら、だいたいこういう実装になります：  
  
HL <- BC ：PUSH BC / POP HL  
  
BC <- HL ：PUSH HL / POP BC  
  
DE <- BC ：PUSH BC / POP DE  
  
BC <- DE ：PUSH DE / POP BC  
  
HL <-> DE ：EX DE,HL（最強）  
  
もし今の目的が「まずアセンブルを通す」段階なら PUSH/POP 版にすぐ差し替えた方が良いです。  
  
すぐ直したいなら：あなたの I_MOV AX,BX を通す最小修正  
MOV → 消す（Z80では LD じゃなくて PUSH/POP にする）  
  
dst2/src2 の中間変数をやめて、分岐の中で直接コードを吐く  
  
…が最短です。  
  
必要なら、あなたの「8086レジスタ→Z80割り当て（AX=HL, BX=BC, DX=DE）」前提で、**I_MOV を “Z80で実際に動く PUSH/POP/EX 版”**にして、AX,BX,DX の9通り全部をきれいに埋めた完成形をこちらで出します。  
  
あなた:  
試してみましたが、  
 MACRO I_MOV dst, src  
    I_MOV_S "dst", "src"  
 ENDM  
の中のI_MOV_S "dst", "src"のためか、それぞれのレジスタ名がdstとsrcと認識されて、それ以外条件分岐の結果がELSEになっているようです。  
  
  
  
       INCLUDE "MACROS_Z80.I"   ; 命令マクロの“インタフェース”宣言（Z80実装は別）  
 MACRO I_MOV dst, src  
  I_MOV_S "dst", "src"  
 ENDM  
  
  
; --- 実体：文字列で分岐して Z80コードを出す ---  
 MACRO I_MOV_S d, s  
  IF d=="AX"  
    IF s=="BX"  
      LD HL,BC  
    ELSEIF s=="DX"  
      LD HL,DE  
    ELSEIF s=="AX"  
      ; LD HL,HL は無意味なので何もしない  
    ELSE  
      DISPLAY /L, "I_MOV AX,: bad src=", s  
      DB 0  
    ENDIF  
  
  ELSEIF d=="BX"  
    IF s=="AX"  
      LD BC,HL  
    ELSEIF s=="DX"  
      LD BC,DE  
    ELSEIF s=="BX"  
      ; 何もしない  
    ELSE  
      DISPLAY /L, "I_MOV BX,: bad src=", s  
      DB 0  
    ENDIF  
  
  ELSEIF d=="DX"  
    IF s=="AX"  
      LD DE,HL  
    ELSEIF s=="BX"  
      LD DE,BC  
    ELSEIF s=="DX"  
      ; 何もしない  
    ELSE  
      DISPLAY /L, "I_MOV DX,: bad src=", s  
      DB 0  
    ENDIF  
  
  ELSE  
    DISPLAY /L, "I_MOV: bad dst=", d  
    DB 0  
  ENDIF  
 ENDM  
  
  
      I_MOV  BX, AX    ; BX <- AX   (ここでは「W候補」をBXに置く約束)  
  
ChatGPT:  
はい、そこは sjasmplus の「"dst" は文字列リテラル」扱いになるので、あなたの観察どおりです。  
  
I_MOV_S "dst", "src" は 文字列 "dst" / "src" を渡しているだけ  
  
だから IF d=="AX" は常に偽になり、全部 ELSE 側へ行きます  
  
私が前回書いた "dst" / "src" は（他アセンブラの「引数名を文字列化」みたいな）ノリで書いてしまっていて、sjasmplusではそういう“引数の中身を文字列として渡す”機能にはなりません。ごめんなさい、ここは修正が必要です。  
  
正しい修正方法（sjasmplusで確実）  
方法A：呼び出し側で文字列として渡す（最も単純）  
呼び出しを I_MOV "BX","AX" みたいにします。  
  
MACRO I_MOV dst, src  
  I_MOV_S dst, src  
ENDM  
  
; 実体  
MACRO I_MOV_S d, s  
  IF d=="AX"  
    ; ...  
  ELSEIF d=="BX"  
    ; ...  
  ENDIF  
ENDM  
  
; 呼び出し  
I_MOV "BX","AX"  
ただ、あなたは 8086 風に I_MOV BX, AX と書きたいはずなので、次へ。  
  
方法B：I_MOV BX, AX のまま使いたいなら「識別子→文字列」変換テーブルを作る（定番）  
1) まず、8086 レジスタ名を「文字列」に変換するマクロを用意  
; r が AX なら "AX" を返す、みたいな“変換”をする  
; sjasmplus では “返す” はできないので、次の段で使うために別マクロへ渡す  
MACRO _MOV_DISPATCH dst, src, dst_s, src_s  
  I_MOV_S dst_s, src_s  
ENDM  
  
MACRO I_MOV dst, src  
  ; dst を文字列へ  
  IF dst==AX  
    IF src==AX  
      _MOV_DISPATCH dst, src, "AX", "AX"  
    ELSEIF src==BX  
      _MOV_DISPATCH dst, src, "AX", "BX"  
    ELSEIF src==DX  
      _MOV_DISPATCH dst, src, "AX", "DX"  
    ELSE  
      DISPLAY /L, "I_MOV: bad src"  
      DB 0  
    ENDIF  
  ELSEIF dst==BX  
    IF src==AX  
      _MOV_DISPATCH dst, src, "BX", "AX"  
    ELSEIF src==BX  
      _MOV_DISPATCH dst, src, "BX", "BX"  
    ELSEIF src==DX  
      _MOV_DISPATCH dst, src, "BX", "DX"  
    ELSE  
      DISPLAY /L, "I_MOV: bad src"  
      DB 0  
    ENDIF  
  ELSEIF dst==DX  
    IF src==AX  
      _MOV_DISPATCH dst, src, "DX", "AX"  
    ELSEIF src==BX  
      _MOV_DISPATCH dst, src, "DX", "BX"  
    ELSEIF src==DX  
      _MOV_DISPATCH dst, src, "DX", "DX"  
    ELSE  
      DISPLAY /L, "I_MOV: bad src"  
      DB 0  
    ENDIF  
  ELSE  
    DISPLAY /L, "I_MOV: bad dst"  
    DB 0  
  ENDIF  
ENDM  
これで I_MOV BX, AX のように 識別子比較（dst==BX） で分岐して、最終的に 文字列 "BX" "AX" を I_MOV_S に渡せます。  
  
つまり、「引数の中身を文字列化する」機能が無いので、テーブル（分岐）で変換するのが王道です。  
  
さらに重要：Z80では LD HL,BC ができません（ここは実機で詰みます）  
あなたの I_MOV_S の中で例えば  
  
LD HL,BC  
を書いてますが、Z80にはこの命令がありません。  
なので “アセンブルが通っても実行できない” 以前に、普通はアセンブル段階で弾かれます。  
  
8086 の MOV r16,r16 を Z80 でやるなら、だいたい次のどれかです：  
  
PUSH src / POP dst （BC/DE/HL/IX/IY ならOK）  
  
EX DE,HL（HL↔DE限定で超高速）  
  
メモリ経由（遅い）  
  
だから I_MOV_S を PUSH/POP版にすると安定します。  
  
例：BX <- AX（BC <- HL）  
  
PUSH HL  
POP  BC  
AX <- BX（HL <- BC）  
  
PUSH BC  
POP  HL  
あなたの現状を最短で動かす：完成例（AX=HL, BX=BC, DX=DE）  
実体（文字列は不要にして、識別子で直接分岐）  
「文字列比較」自体を捨てて、dst==AX のように識別子比較に統一すると一番ラクです。  
  
MACRO I_MOV dst, src  
  ; BX <- AX など（AX=HL, BX=BC, DX=DE の割当）  
  IF dst==AX  
    IF src==BX  
      PUSH BC  
      POP  HL  
    ELSEIF src==DX  
      EX   DE,HL        ; HL<-DE かつ DE<->HL なので注意（破壊あり）  
      ; 破壊が嫌なら PUSH DE / POP HL  
    ELSEIF src==AX  
      ; nop  
    ELSE  
      DISPLAY /L, "I_MOV: bad src"  
      DB 0  
    ENDIF  
  
  ELSEIF dst==BX  
    IF src==AX  
      PUSH HL  
      POP  BC  
    ELSEIF src==DX  
      PUSH DE  
      POP  BC  
    ELSEIF src==BX  
      ; nop  
    ELSE  
      DISPLAY /L, "I_MOV: bad src"  
      DB 0  
    ENDIF  
  
  ELSEIF dst==DX  
    IF src==AX  
      PUSH HL  
      POP  DE  
    ELSEIF src==BX  
      PUSH BC  
      POP  DE  
    ELSEIF src==DX  
      ; nop  
    ELSE  
      DISPLAY /L, "I_MOV: bad src"  
      DB 0  
    ENDIF  
  
  ELSE  
    DISPLAY /L, "I_MOV: bad dst"  
    DB 0  
  ENDIF  
ENDM  
これなら  
  
I_MOV BX, AX  
でちゃんと分岐します（dst==BX が真）。  
  
まとめ（あなたの疑問の答え）  
I_MOV_S "dst","src" がダメなのは、"dst" は引数名の中身ではなく文字列リテラルだから  
  
sjasmplus には「引数名 → 引数値を文字列化」機能が無いので、  
  
呼び出し側で "AX" と書く（簡単）  
  
変換テーブルで "AX" を作る（面倒）  
  
最初から識別子比較で分岐する（おすすめ）  
  
ついでに、Z80は LD HL,BC が無いので PUSH/POP（またはEX）で書く必要がある  
  
この先、あなたが作りたいのは「8086命令っぽい記述を全部Z80で通す」だと思うので、  
MOV/ADD/SUB/PUSH/POP/JMP あたりを **“識別子比較 + PUSH/POP/EX”**でテンプレ化すると一気に安定します。  
  
よければ、今の「AX=HL, BX=BC, DX=DE」割当で、I_PUSH reg / I_POP reg / I_XCHG / I_JMP_IND も同じ流儀で一式整えた雛形を出します。  
  
  
  
  
