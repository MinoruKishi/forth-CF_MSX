# GitHub MinoruKishi/forth-CF_MSX   
  
## CCurl/CFを初めて読んだ人がChatGPTの力を借りてのMSX0向けFORTH改造の顛末記  
  
#   
  
# あなた:  
  
20260109_cf_test01.zip  
  
Zip アーカイブ    
添付したファイルはcfというforth言語システムです。これからZ80をベースとするforth言語システムを開発します。最適のZ80用の開発環境とソースコードを教えてください。  
  
# ChatGPT:  
  
思考時間: 2m 0s  
  
了解です。添付の cf は「3つの小さい C ソースで動く、ColorForth/Tachyon 影響の小型 Forth」なので（README にもその方針が明記されています）、この“最小カーネル＋拡張を Forth ソースで積む”流れは Z80 でもかなり相性が良いです。    
ここでは「Z80 ベースの Forth を新規開発」する前提で、**おすすめの開発環境（実用優先）**と、出発点として強い既存ソースをまとめます。  
  
---  
  
## **1) 最適な Z80 開発環境（まずこれが鉄板）**  
  
### **A. クロス開発（PC上でビルド→エミュレータ/実機へ転送）**  
  
* z88dk：Z80/8080 系の開発キット（C＋ASM混在もOK、ターゲット多数）。Z80 のツール群を一式まとめて使えるのが強いです。     
* SjASMPlus：モダンな Z80 クロスアセンブラ。ドキュメントも整っていて、Windows/Linux/macOS で使えます。     
* エミュレータ（CP/M系なら特に便利）    
  * z80pack：8080/Z80 複数システムの成熟したエミュレータ。CP/M 環境も含めて回しやすいです。   
  
この構成が良い理由    
Forth を作るときは「辞書」「スレッディドコード」「I/O」が肝で、デバッグ回数が多いです。まず PC 上で回せるエミュレーションがあると開発速度が段違いになります。  
  
---  
  
### **B. 実機で育てる（おすすめの入門ハード）**  
  
* RC2014：シリアル端末で運用しやすい、Z80 自作系で定番。     
* Z80-MBC2：SD で “ディスク” を扱えて、CP/M 2.2/3 なども動かせる系。Forth を育てる土台として強いです。   
  
---  
  
## **2) そのまま使える「出発点の Forth ソースコード」候補**  
  
あなたの cf の方向性（小さい核・作りやすい VM/辞書）に近い順に挙げます。  
  
### **候補1：CamelForth/Z80（ANSI寄り・古典の良さが詰まってる）**  
  
* 公式の CamelForth/Z80（Brad Rodriguez）     
* RC2014 向けに整備された移植（ビルドに z88dk を使う例）   
  
おすすめポイント：    
「辞書・インタプリタ・コンパイラ」が一通り揃っていて、Z80 Forth の“正攻法”を学びながら改造できます。  
  
---  
  
### **候補2：eForth Z80（最小核で始めたいなら最有力）**  
  
* eForth の Z80 実装（EFZ80.ASM がある）   
  
おすすめポイント：    
小さく立ち上がるので、cf の“ミニマル志向”に合います。まず動く核を作り、その上にあなた流の語彙・OPCODE設計を積み上げやすいです。  
  
---  
  
### **候補3：Firth（ROM 常駐寄り・組込み向けに触りやすい）**  
  
* “Z80 の ROM で動かす”方向の Forth 実装   
  
---  
  
## **3) あなたが「cf を持っている」ことを活かす作り方（現実的ルート）**  
  
cf の README にある通り、cf は **「OPCODE列」＋「プリミティブ」＋「辞書」**の発想が強いタイプです。Z80 化するなら、次の順が最短です。  
  
1. ターゲットを決める（CP/M上か、裸（ROM/RAM直叩き）か）    
   * CP/M 上：I/O が楽、デバッグが楽（z80pack 等が効く）     
   * 裸：最終的に“あなたの機械”になる。が、最初のI/Oで詰まりやすい    
2. 最小カーネルを Z80 ASM で作る    
   * データスタック／リターンスタック    
   * NEXT ループ（スレッディドコード実行）    
   * 最小 I/O（KEY/EMIT 相当）    
3. 高レベル語は Forth ソースで増やす（cf-boot.fth 的に）    
4. 慣れたら cf の “OPCODE/プリミティブ設計”を Z80 に合わせて最適化  
  
このとき、CamelForth や eForth は「最小核・辞書の形・インタプリタ」が既にあるので、全部自作より早く “あなたの cf 的設計”へ寄せていけます。  
  
---  
  
## **まず入れるべきセット（結論）**  
  
* 開発ツール：z88dk + SjASMPlus     
* 実行環境（最初）：z80pack（CP/M エミュ）     
* ベースソース：    
  * まず堅実に：CamelForth/Z80     
  * 最小核から：eForth Z80   
  
---  
  
必要なら、添付の cf（zip内の cf.c / system.c / cf-boot.fth など）をこちらで読み解いて、    
**「Z80版で同じ構造にするなら、どのC関数/概念がZ80のどのルーチン/テーブルに対応するか」**を対応表にして、最小カーネル設計（メモリマップ、辞書形式、NEXT 方式、プリミティブ表）まで一気に具体化します。  
  
# あなた:  
  
cfはforth-dwcから発展した、独立したforthシステムです。cfは基本的な機能しかサポートしていませんが、開発を進める中で検討していく予定です。このcfが動作する環境はCPUがZ80であるMSX0になります。cfのアプリケーションはFORTH言語で記述します。場合によってはcf内のC言語で書かれたソースコードを修正する場合もあります。核となるコードはZ80のアセンブラで記述します。方針としては以下のようになります。    
開発ツール：z88dk + SjASMPlus  
  
実行環境（最初）：MSX0 （もしくはMSXエミュレータを使用する）  
  
ベースソース：cfのZIPファイル内のアセンブラ、Cソースコード、FORTHソースコード  
  
# ChatGPT:  
  
思考時間: 1m 48s  
  
いいですね、その方針（Forthアプリは cf のForthで書く／必要なら C を触る／核は Z80 ASM）は、cf の設計（“OPCODE列＋プリミティブ表”）と噛み合っています。  
  
ただ最初に一点だけ事実確認：添付ZIPの中には Z80 アセンブラの核コードは入っていません（cf.c / system.c / cf-boot.fth / makefile などの “PC上で動く cf” 一式です）。なので MSX0(Z80)版の核はこれから新規に作ることになります（方針どおりでOKです）。  
  
以下、あなたの前提（z88dk + SjASMPlus、MSX0/エミュ）に合わせて、開発環境の組み方と cf をMSX0へ落とす具体的な手順を、最短ルートで書きます。  
  
---  
  
## **1) 実行環境：MSX0 か openMSX（最初は openMSX 推奨）**  
  
* openMSX はコンソール/デバッガが強く、Z80のステップ実行・ブレーク・メモリ/スロット確認がしやすいです。    
* エディタ＋VSCode派なら、DeZog（VSCodeのZ80デバッガ）＋openMSX という組み合わせもよく使われます。  
  
結論：最初は openMSXで核を育ててからMSX0実機がいちばん速いです（I/Oやメモリの事故がすぐ見える）。  
  
---  
  
## **2) 出力形式を決める：ROM起動か、MSX-DOS(COM)か**  
  
どちらでも行けますが、立ち上げ速度は違います。  
  
### **A. ROM（カートリッジ）形式**  
  
* z88dk には MSX ROM 用のスタートアップやROM構成があり、参考になります。    
* SjASMPlus でも ROM バイナリ生成は定番です。  
  
### **B. MSX-DOS の .COM（ディスク起動）**  
  
* 既存ノウハウが多く、エミュ上でテストが楽なことが多いです（.dskに入れて実行）。  
  
迷ったら：最初は MSX-DOS .COM（デバッグが楽）→固まったら ROM化 が無難です。  
  
---  
  
## **3) cf を MSX0(Z80)へ移植する“骨組み”の作り方**  
  
cf の README にある核心はこれです：  
  
* 「プログラム＝OPCODE列」    
* 「OPCODEがプリミティブ番号か、LITか、XT（CODEスロット）かで解釈」    
  （この設計が Z80 上の VM/インタプリタに落としやすい）  
  
そこで移植は 3層に分けると綺麗に進みます。  
  
### **層1：Z80 “核”(VMランタイム) — ここがあなたのZ80 ASM担当**  
  
最低限これだけ先に作ると、cf-boot.fth 相当を載せられます。  
  
* データスタック（cell幅を先に決める：16bitか32bitか）    
* リターンスタック    
* NEXT ループ（OPCODEフェッチ→ディスパッチ）    
* LIT の処理    
* 最小I/O（key/emit 相当。まずは BIOS/BDOS 呼び出しでOK）  
  
### **層2：プリミティブ表（ディスパッチ先）**  
  
cf.c のプリミティブは「番号が連番」の前提なので、MSX版でも  
  
* 同じ番号＝同じ意味 に揃えるのが後々ラクです（Forthソース互換が保てる）    
* まずは cf-boot.fth を通すのに必要な最小セットだけ実装→増やす  
  
### **層3：高レベル（Forth）— ここがアプリ/拡張**  
  
* cf-boot.fth を “MSX版ブート” にしていく    
* アプリ（あなたのFORTHコード）もここに積む  
  
---  
  
## **4) ツールチェーンの具体：z88dk + SjASMPlus の役割分担**  
  
あなたの方針どおりでOKです。おすすめの分け方はこれ：  
  
* SjASMPlus：核VM（Z80 ASM）、ROMヘッダ/スロット初期化、超低レベルI/O    
  （SjASMPlus のドキュメントも新しめがまとまっています）    
* z88dk：補助C（必要なら）、リンク、ターゲット固有crt0の流用/研究    
  MSXターゲットのROM/起動コード例も参照できます。  
  
※「Cは使わない」でもOKですが、**最初の“動く確認”**に z88dk のテンプレやcrt0がヒントになります。  
  
---  
  
## **5) リポジトリ構成（おすすめ）**  
  
今のZIPは “PC版cf” なので、そのまま活かしつつ MSX0版を並走させると安全です。  
  
~~~
  
cf/  
  |  
  --  host_pc/        # 既存ZIPの cf.c, system.c, cf-boot.fth（PCで動く参照実装）  
  |  
  --  target_msx/  
        |    
        --  asm/          # Z80核（NEXT, stacks, prims最低限）  
        |  
        --  forth/        # msx用 boot とアプリ  
        |  
        --  tools/        # 生成スクリプト（後述）  
        |  
        --  build/        # 出力 .COM （今回はsjasmplusを使用するので、.ROMではありません。）    
~~~
---  
  
## **6) まずやるべき “最初のマイルストーン”（詰まりにくい順）**  
  
1. cell幅を決める（16bit推奨で開始：Z80で速い。必要なら32bitへ拡張）    
2. openMSX で “key→emitのエコー” が動く最小 .COM or ROM を作る    
3. その上で VM の NEXT＋LIT＋数個のプリミティブ を実装    
4. cf-boot.fth を MSX側で読み込む方式を作る（最初は固定領域に同梱でもOK）    
5. 以降、プリミティブとForth語彙を増やす  
  
---  
  
## **7) ここから先、私が具体化できる作業（次の一手）**  
  
あなたのZIPの中身（cf.c/cf.h/cf-boot.fth/README）を基準に、  
  
* MSX版で最初に必要になるプリミティブ最小セット（cf-boot.fth を通すのに必要なもの）を抽出して    
* **Z80側のディスパッチ表（番号表）**と    
* 最小VMのメモリマップ案（辞書、HERE、スタック領域、文字列領域）  
  
を、実ファイルに即して書けます。  
  
そのために次のステップとして、ZIP内の cf-boot.fth と cf.h の “メモリ/辞書/セル幅” に関わる部分をこちらで読み取り、MSX0向けの最小核設計に落とします（あなたは「核はZ80 ASM」なので、Z80のルーチン雛形まで出せます）。  
  
必要ならこのまま進めます。  
  
# あなた:  
  
「６) まずやるべき “最初のマイルストーン”（詰まりにくい順）  
  
１．cell幅を決める（16bit推奨で開始：Z80で速い。必要なら32bitへ拡張）  
  
２．openMSX で “key→emitのエコー” が動く最小 .COM or ROM を作る」    
cel幅は16ビットで行きましょう。その上でCOMファイルを作成するための各ファイルのサンプルを作成できますか？  
  
# ChatGPT:  
  
思考時間: 47s  
  
はい、できます。ここでは cell幅=16bit 前提で、まずは MSX-DOS の .COM として動く「キー入力→そのまま出力（エコー）」の最小サンプル一式を作ります。  
  
MSX-DOS の .COM は 0x0100 にロードされ実行されるのが前提で、BDOSコールは **CALL 0005h（関数番号はCレジスタ）**で呼び出します。  
  
---  
  
# **1) ファイル構成（最小）**  

~~~
msx_cf_skel/  
     |  
     --  src/  
            |  
            --  echo.asm  
            |  
            --  Makefile          (Linux/macOS向け。Windowsなら build.bat でもOK)  
            |  
            --  build.bat         (Windows向け)  
            |  
            --  README.md    
~~~
---  
  
# **2) src/echo.asm（SjASMPlus・MSX-DOS .COM）**  
  
* ORG 0x0100    
* BDOS 0x06（Direct Console I/O）を使い、    
  * E=0xFF で キー入力（エコー無し）    
  * E=文字 で コンソール出力    
* ESC(0x1B)で終了（BDOS 0x00）    
  ※ BDOS呼び出しの仕方は MSX-DOS では CALL 0005h が基本です。  
  
; echo.asm - Minimal MSX-DOS .COM: key -> emit (echo)  
  
; Assemble: sjasmplus --raw=echo.com src/echo.asm  
  
; Run on MSX-DOS: A:\>ECHO  
~~~  
       DEVICE ZXSPECTRUM48    ; (SjASMPlusの機能。無くても可)  
  
       ORG     0100h  
  
BDOS    EQU     0005h  
  
; BDOS functions (CP/M compatible in MSX-DOS)  
  
F_TERM0 EQU     00h    ; terminate program  
  
F_DIRIO EQU     06h    ; direct console I/O (E=FFh input, else output)  
  
start:  
  
main_loop:  
  
       ; --- read key (no echo): A = char, 0 if none (implementation dependent)  
  
       LD      C, F_DIRIO  
  
       LD      E, 0FFh  
  
       CALL    BDOS  
  
       OR      A  
  
       JR      Z, main_loop      ; no key => poll again  
  
       CP      1Bh               ; ESC to exit  
  
       JR      Z, exit  
  
       ; --- emit char  
  
       LD      E, A  
  
       LD      C, F_DIRIO  
  
       CALL    BDOS  
  
       JR      main_loop  
  
exit:  
  
       LD      C, F_TERM0  
  
       CALL    BDOS  
~~~
  
補足：BDOS 0x06 の挙動（入力ポーリング等）は環境で差が出ることがあるので、まずはこれで「キー→表示」が動くことを確認し、次段で BIOS の CHGET/CHPUT などに切り替えるのが安定です。CHGET/CHPUT の存在自体は MSX BIOS の定番ルーチンです。  
  
---  
  
# **3) build.bat（Windows向け）**  
  
~~~
@echo off  
  
setlocal  
  
rem sjasmplus.exe が PATH にある前提。無い場合はフルパスに。  
  
if not exist build mkdir build  
  
sjasmplus --raw=build\echo.com src\echo.asm  
  
if errorlevel 1 (  
  
 echo Build failed.  
  
 exit /b 1  
  
)  
  
echo Built: build\echo.com  
  
endlocal    
~~~
---  
  
# **4) Makefile（Linux/macOS向け・任意）**  
  
~~~
SJASMPLUS ?= sjasmplus  
  
all: build/echo.com  
  
build/echo.com: src/echo.asm  
  
	@mkdir -p build  
  
	$(SJASMPLUS) --raw=$@ $<  
  
clean:  
  
	rm -rf build    
~~~
---  
  
# **5) README.md（動かし方）**  
  
# MSX-DOS COM minimal echo sample (SjASMPlus)  
  
## Build (Windows)  
  
- run `build.bat`  
  
- output: `build/echo.com`  
  
## Build (Linux/macOS)  
  
- `make`  
  
## Run  
  
- Copy `echo.com` to an MSX-DOS disk image (.dsk) or to your MSX0 storage  
  
- On MSX-DOS: `A:\>ECHO`  
  
- Press keys -> characters appear  
  
- Press ESC to exit    
---  
  
# **次（cfへ繋げるための“最小骨格”）**  
  
この echo.com が動いたら、次は **16bit cell のスタック（DSP/RSP）**と NEXT（ディスパッチ） を入れて、LIT と数個プリミティブだけ実装していくのが王道です。  
  
もしよければ次のターンで、  
  
* 16bit cell のデータスタック／リターンスタックの配置案（メモリマップ）    
* NEXT ループの最小形（Z80）    
* cf の cf-boot.fth を動かすのに必要な 最小プリミティブ群の洗い出し  
  
まで、具体的なファイル雛形として続けて出します。  
  
# あなた:  
  
UnderstandingFORTH_FIG79_MEMO.zip  
  
Zip アーカイブ    
では、私が最初に学んだFORTHー79で書かれた８０８６用ソースコードを添付します。  
  
最初に行いたいのは８０８６の機械語のコードをそれぞれをマクロ名に変更してください。各マクロには実行部分をZ80コードで埋めます。    
具体例としては、次のアセンブラコードにある機械語命令をそれぞれのマクロに変更します。  
  
~~~
; ****FORTH_INNER_INTERPRETER****  
  
DPUSH: PUSH DX    
APUSH: PUSH AX    
NEXT:  LODSW     ; AX <- (IP)    
       MOV BX,AX ; IP <- IP+2    
NEXT1: MOV DX,BX    
       INC DX    ; SET W    
       JMP WORD PTR [BX] ; JUMP TO (IP)  
  
; ****FORTH_DICTIONARY****  
  
; LIT  
  
       DB 83H    
       DB 'LI'    
       DB 'T'+80H    
       DW 0 ; end of dictionary    
LIT:   DW $+2    
       LODSW    
       JMP APUSH  
~~~  
これをもとにして、例えば    
 　'MOV BX,AX ; IP <- IP+2'    
の部分も一つのマクロを呼び出して実行するように書き換えます。その時は、MOVがマクロ名でAX,BXがそのマクロの変数に相当します。今回はZ80で実行するのでZ80用のコードを記述します。マクロの定義部分は別ファイルにしておけば、後で別のCPUでも同じことをするときに便利な気がします。    
レジスタ名は、私はAXやBXで理解しているのでそちらのほうがわかりやすいのですが、やっぱりFORTHでの呼び方を用いるとＩＰ，ＳＰ，ＲＰ，Ｗのようなマクロ内の変数名のほうがいいと思います。    
「標準ＦＯＲＴＨ　井上外志雄　著　共立出版株式会社」の180ページにＺ８０の場合と６８０９の場合の例がありますので、メモしておきます。  
  
 ◎Ｚ８０の場合    
~~~
  IP = BC    
  SP = SP    
  RP = IX    
  W  = DE    
~~~
~~~
NEXT = (IY)  
  
NEXT:   LD A,(BC)    
        INC BC    
        LD L,A    
        LD A,(BC)    
        INC BC    
        LD H,A    
NEXT1:  LD E,(HL)    
        INC HL    
        LD D,(HL)    
        EX DE,HL    
        JP (HL)  
  
DOCOL:  DEC IX    
        DEC IX    
        LD (IX+0),C    
        LD (IX+1),B    
        INC DE    
        LD C,E    
        LD B,D    
        JP (IY)  
  
SEMIS:  DW $+2    
        LD C,(IX+0)    
        LD B,(IX+1)    
        INC IX    
        INC IX    
       JP (IY)  
~~~
    
◎６８０９の場合    
~~~
  IP = Y    
  SP = U    
  RP = S    
  W  = X  
~~~
~~~  
NEXT   LDX ,Y++    
       JMP [,X]  
  
DOCOL  PSHS Y    
       LEAY 2,X    
       BRA NEXT  
  
SEMIS  FDB \*+2    
       LDY ,S++    
       BRA NEXT  
~~~
    
# ChatGPT:  
  
思考時間: 1m 23s  
  
できます。まずは **「8086の命令を“マクロ呼び出し”に置き換えたソース」**と、別ファイルに **「そのマクロをZ80コードで実装した定義」**を用意します。    
（最初はご提示の *INNER INTERPRETER と LIT* 周辺だけを対象にします。残りの辞書語彙も同じやり方で増やせます。）  
  
ここで重要なのは **“8086のレジスタ名（AX/BX…）のまま書ける”**ようにしつつ、実体は Z80でのForthレジスタ割り当てに落とすことです。  
  
---  
  
## **0) Z80側のレジスタ割り当て（あなたのメモの方式で固定）**  
  
あなたのメモ（標準FORTH p.180）に合わせます：  
  
* IP = BC    
* SP = SP    
* RP = IX    
* W = DE    
* NEXT = (IY) （IYを“NEXTへ飛ぶアドレス”として使う流儀）  
  
さらに、Z80実装でよく使う都合上：  
  
* HL は 一時レジスタ/XT取り出しに使う（NEXTの定番形）  
  
---  
  
## **1) 8086コードを「命令＝マクロ」に置換したファイル例**  
  
ファイル名例：src/inner_8086_macroified.asm  
  
ここは 8086風の見た目を残しますが、実体は “マクロ呼び出し” だけになります。  
  
~~~
; inner_8086_macroified.asm  
  
; 8086の命令をすべてマクロ呼び出しに置換した版（Z80へ落とす前段）  
  
       INCLUDE "macros_iface.inc"   ; 命令マクロの“インタフェース”宣言（Z80実装は別）  
  
; ****FORTH_INNER_INTERPRETER****  
  
DPUSH:  I_PUSH DX  
  
APUSH:  I_PUSH AX  
  
NEXT:   I_LODSW          ; AX <- (IP) , IP <- IP+2  
  
       I_MOV  BX, AX    ; BX <- AX   (ここでは「W候補」をBXに置く約束)  
  
NEXT1:  I_MOV  DX, BX  
  
       I_INC  DX        ; SET W  
  
       I_JMP_IND BX     ; JMP WORD PTR [BX]  
  
; ****FORTH_DICTIONARY****  
  
; LIT  
  
       DB 83H  
  
       DB 'LI'  
  
       DB 'T'+80H  
  
       DW 0  
  
LIT:    DW $+2  
  
       I_LODSW  
  
       I_JMP APUSH  
~~~
  
ポイント：  
  
* PUSH / MOV / INC / JMP みたいな 個々の命令が全部マクロです。    
* LODSW は8086固有なので、これも I_LODSW というマクロにします（Z80で “(IP)の16bit読み→IP+=2” を実装）。  
  
---  
  
## **2) マクロ定義ファイル（Z80実装）例**  
  
ファイル名例：src/macros_z80.inc（SjASMPlus想定）  
  
ここでは、上の I_PUSH AX といった “8086名” を受け取り、実際には Z80のIP/W/RP… に割り当てたコードを吐きます。  
  
### **2-1) まず「8086レジスタ名」をZ80上の実体に対応させる**  
  
ここが肝です。最初はこの対応で進めるのが安全です：  
  
* AX … HL（TOS/作業用。LITで値を受けるのもHLに）    
* BX … HLまたはDEにしたくなるが、NEXTの定番を崩したくないので、**“BXはHL扱い”**に寄せる    
* DX … DE（W）（あなたのメモ通り）  
  
つまりこの段階では「AX/BX/DX」という“名前”は残しつつ、Z80では HL/DE中心で動かします。  
  
### **2-2) macros_z80.inc**  
  
~~~
; macros_z80.inc (SjASMPlus)  
  
; 8086命令マクロをZ80コードで実装する  
  
; --- Forth register assignment (fixed) ---  
  
; IP = BC  
  
; RP = IX  
  
; W  = DE  
  
; NEXT vector = IY (IY points to routine address or is used as JP (IY) target)  
  
;  
  
; Convention for 8086-like names:  
  
;   AX -> HL  
  
;   DX -> DE  
  
;   BX -> HL (we treat BX as HL to keep NEXT idiom simple)  
  
; ========== helpers ==========  
  
MACRO I_JMP label  
  
   JP label  
  
ENDM  
  
MACRO I_JMP_IND reg  
  
   ; 8086: JMP WORD PTR [reg]  
  
   ; Z80:  JP (regpair)  ※regは HL を想定（BX=HL）  
  
   JP (reg)  
  
ENDM  
  
MACRO I_INC reg  
  
   ; 8086: INC DX など  
  
   INC reg  
  
ENDM  
  
MACRO I_MOV dst, src  
  
   ; 8086: MOV DX,BX / MOV BX,AX など  
  
   ; Z80: 16bitレジスタ同士を想定（HL/DE/BC/IX など）  
  
   LD dst, src  
  
ENDM  
  
MACRO I_PUSH reg  
  
   ; 8086: PUSH AX/DX  
  
   ; Z80: PUSH HL/DE  
  
   PUSH reg  
  
ENDM  
  
MACRO I_POP reg  
  
   POP reg  
  
ENDM  
  
; 8086: LODSW  (AX <- (IP), IP <- IP+2)  
  
; Z80:  (IP=BC)から16bitを読み、HL(=AX)へ。BCを+2  
  
MACRO I_LODSW  
  
   LD A,(BC)  
  
   INC BC  
  
   LD L,A  
  
   LD A,(BC)  
  
   INC BC  
  
   LD H,A  
  
ENDM    
~~~
---  
  
## **3) ここまでで「あなたがやりたい形」になっています**  
  
* inner_8086_macroified.asm は “命令が全部マクロ” になっている    
* macros_z80.inc を差し替えれば、別CPUへも同じ“命令列”を移植しやすい    
* そして、あなたが言う通り 命令マクロ定義だけ別ファイルなので、CPU差分はそこに閉じ込められます  
  
---  
  
## **4) 次にやると良いこと（あなたのメモのNEXT/DOCOL/SEMISに合わせる）**  
  
いまの I_LODSW や I_JMP_IND は “命令単位” です。    
一方で、Forthの性能と分かりやすさのためには、最終的に  
  
* NEXT（定番のZ80 NEXT）    
* DOCOL    
* SEMIS  
  
は **命令マクロではなく“語（word）マクロ”**としても別途用意した方が楽です。  
  
あなたのメモにあるZ80版NEXTは、まさにその「語マクロ」側の典型です。  
  
---  
  
## **5) 添付ZIPの8086ソースをこの方式で“自動で”置換することも可能**  
  
あなたが添付した UnderstandingFORTH_FIG79_MEMO.zip の 4TH_dos.txt には、まさにこの DPUSH/APUSH/NEXT/LIT が入っています。    
なので次は、  
  
* 4TH_dos.txt の該当範囲を私が読み取り    
* 命令→I_マクロに置換した「続き（EXEC/BRANCH/0BRANCH…）」を生成  
  
までまとめて作れます。  
  
---  
  
もし次に進めるなら、どちらが良いですか？  
  
1. まずはこのサンプルを openMSX + MSX-DOS(COM) でビルドして動作確認する（最低限の起動枠を作る）    
2. 先に 4TH_dos.txt の 辞書語彙（EXEC/BRANCH/0BRANCH…）も一括でマクロ化して、Z80側の実装を埋めていく  
  
（どちらでも、こちらで“そのまま使えるファイル一式”を出せます。）  
  
# あなた:  
  
以下のディレクトリで、sjasmplusを実行しましたが、エラーになってしまいます。何が悪かったんでしょう？  
  
~~~
　　src  ---- --test_echo.asm    
         build  -- --    
         include  --MACROS_Z80.I  
  
$ sjasmplus --inc --inc=include --raw=test_echo.com src/test_echo.asm    
SjASMPlus Z80 Cross-Assembler v1.21.1 (https://github.com/z00m128/sjasmplus)    
Pass 1 complete (0 errors)    
Pass 2 complete (0 errors)    
test_echo.asm(6): error: Unrecognized instruction: MACROS_Z80.I    
Pass 3 complete    
Errors: 1, warnings: 0, compiled: 38 lines, work time: 0.001 seconds  
~~~
  
# ChatGPT:  
  
原因はほぼこれです：  
  
INCLUDE "MACROS_Z80.I" の書き方（または拡張子）が、SjASMPlus の INCLUDE 文として解釈されていないため、6行目の MACROS_Z80.I という文字列が「命令」として読まれてしまっています。  
  
ログの  
  
Unrecognized instruction: MACROS_Z80.I  
  
は、「その行が INCLUDE ... ではなく、単に MACROS_Z80.I という“命令っぽいもの”が書かれている」と SjASMPlus が判断した時に出ます。  
  
---  
  
## **直し方（いちばん確実）**  
  
### **✅ 1)** test_echo.asm **の6行目をこうしてください**  
  
SjASMPlus は INCLUDE が必要です（include だけ書いてもダメ）。  
  
   INCLUDE "MACROS_Z80.I"  
  
* 行頭にラベル等を置かない    
* INCLUDE を必ず書く    
* ファイル名は引用符で囲む  
  
---  
  
## **✅ 2) ついでに “拡張子を** .inc **にする”のもおすすめ**  
  
MACROS_Z80.I だと「ヘッダっぽい」ですが、SjASMPlus 的には .inc が一般的で事故が減ります。  
  
include/MACROS_Z80.inc に改名して、  
  
   INCLUDE "MACROS_Z80.inc"  
  
にすると安全です。  
  
---  
  
## **✅ 3)** --inc **オプションの使い方も整理（あなたのコマンドはOK寄り）**  
  
あなたの  
  
sjasmplus --inc --inc=include --raw=test_echo.com src/test_echo.asm  
  
は「include ディレクトリを include path に追加する」意味なので方向性は合っています。  
  
ただし、SjASMPlus のバージョンやビルドにより --inc の解釈が微妙に違うことがあるので、より確実な書き方はこれです（両方書いてもOK）：  
  
sjasmplus --inc=include --raw=test_echo.com src/test_echo.asm    
---  
  
## **よくある “同じエラー” の別原因（念のため）**  
  
次も当てはまると同じ症状になります。  
  
* INCLUDE の前に ; が付いていてコメントになっている    
* ; INCLUDE "MACROS_Z80.I"   ← これだと読み込まれない    
* MACROS_Z80.I              ← 次の行が命令扱いになって落ちる    
* INCLUDE を INCLUDES や #include と書いている（SjASMPlusではダメ）    
* 全角スペースが混じっていて INCLUDE と認識されていない（稀）  
  
---  
  
## **すぐ動く最小例（6行目付近の形）**  
  
src/test_echo.asm の冒頭をこうしてみてください：  
  
~~~
       ORG 0100h  
  
       INCLUDE "MACROS_Z80.I"  
  
BDOS    EQU 0005h  
  
; ...    
~~~
---  
  
もし src/test_echo.asm の先頭10行だけ貼ってくれたら、どこが「INCLUDE として認識されていない」かをピンポイントで直した版を返せます。  
  
[(次へ進む)](./CF-MSX_09_a.md)      
[(README.mdへ戻る)](../README.md)    
  
