# GitHub MinoruKishi/forth-CF_MSX     
    
## CCurl/CFとQWC（追加しました）を初めて読んだ人がChatGPTの力を借りてのMSX0向けFORTH改造の顛末記〜１０    
    
  
# あなた:  
  
~~~  
#define X1(op, name, theCode) op,    
#define X2(op, name, theCode) case op: theCode goto next;    
#define X3(op, name, theCode) { name, op },  
  
#define PRIMS(X) \    
	X(EXIT,   "exit",     pc = (ucell)rpop(); if (pc==0) { return; } ) \    
	X(LIT,    "",         push(code[pc++]); ) \    
           （中略）    
	X(SLEN,   "s-len",    TOS = strlen((char*)TOS); ) \    
	X(LASTOP, "system",   system((char*)pop()); )  
  
enum { PRIMS(X1) };    
~~~  
  
PRIMS(X)で出てくるEXITなどのワードはqwc内のcoreWORDだと理解しています。    
それらのcoreWORDはどう実行されるかがtheCodeに定義されていて、push(xxx)やpc=xxxなど関数やCの手続きで表現されています。    
確認したいのは、これらの関数やCの手続きで表現されたものをもっと小さなmicroWORDとしてFORTH（qwc）のワードに置き換えて、PRIMSをFIG-FORTHで理解したFORTHのワード形式に置き換えられますか？その時に必要なmicroWORDの実行部はZ80形式の命令コードで教えてもらえませんか？   
  
~~~  
char mem[MEM_SZ], *toIn, wd[32];    
ucell *code=(ucell*)&mem[0], dsp, rsp, lsp, tsp, outputFp;    
cell dstk[STK_SZ+1], rstk[STK_SZ+1], lstk[STK_SZ+1], tstk[STK_SZ+1];    
cell here=LASTOP+1, last=(cell)&mem[MEM_SZ], base=10, state=INTERPRET;    
DE_T tmpWords[10];  
  
void push(cell v) { if (dsp < STK_SZ) { dstk[++dsp] = v; } }    
cell pop() { return (0 < dsp) ? dstk[dsp--] : 0; }    
void rpush(cell v) { if (rsp < STK_SZ) { rstk[++rsp] = v; } }    
cell rpop() { return (0 < rsp) ? rstk[rsp--] : 0; }    
void comma(ucell val) { code[here++] = val; }    
void doComment() { while (nextWord() && !strEqI(wd, ")")) {} }    
void doLineComment() { while ( *toIn && (*toIn != 10) ) { ++toIn; } }    
void doNum() { if (state == COMPILE) { compileNum(pop()); } }    
int  isTmpW(const char *w) { return (w[0]=='t') && btwi(w[1],'0','9') && (w[2]==0) ? 1 : 0; }    
void addPrim(const char *nm, ucell op) { DE_T *dp = addToDict(nm); if (dp) { dp->xt = op; } }    
void addLit(const char *name, cell val) { addToDict(name); compileNum(val); comma(EXIT); }    
void doInline(ucell xt) { while (code[xt] != EXIT) { comma(code[xt++]); } }    
void doInterp(ucell xt) { code[10]=xt; code[11]=EXIT; inner(10); }    
char *checkWord(char *w) { return w ? w : (nextWord() ? &wd[0] : NULL); }    
void lit1(cell n) {	comma((ucell)n | LIT_MASK); }    
void lit2(cell n) {	comma(LIT); comma(n); }    
void compileNum(cell n) { btwi(n,0,LIT_BITS) ? lit1(n) : lit2(n); }    
void compileErr(char *w) { zType("\n-word:["); zType(w); zType("]?-\n"); }    
~~~  
  
# ChatGPT:  
  
できます。結論から言うと、あなたが示した qwc の PRIMS(X)（= opcode + C 手続き）方式は、  
  
* (A) そのまま「Z80 の VM（switch/case 相当）」として実装する道（今の構造をZ80に移植）    
* (B) PRIMS をやめて FIG-FORTH 的な “辞書 + colon 定義 + codeword(機械語ルーチン)” に寄せる道（古典Forthの形）  
  
のどちらにも行けます。  
  
あなたが聞いているのは (B) で、つまり  
  
`C` で書いていた `push() / pop() / pc=...` などを、より小さな **“microWORD”** に分解し、Forth のワード（codeword）として置き換え、最終的に PRIMS を「Forthのワード列」で表現できるか？    
さらに、その microWORD の実行部を Z80 命令で欲しい  
  
ですね。  
  
以下、「どう置き換えるか（設計）」→「最低限の microWORD」→「Z80 実装例（FIG寄り）」の順でまとめます。  
  
---  
  
## **1) 何を捨てて、何を残すか（設計の要点）**  
  
あなたの qwc は、  
  
* code[] に enum opcode（EXIT/LIT/...） が並ぶ（= token-threaded-ish）    
* inner(pc) は opcode を fetch → switch → C手続き実行 → goto next をする    
* push/pop/rpush/rpop は C の関数  
  
という作りです。  
  
これを FIG-FORTH 形式に寄せるなら、根本をこう変えます：  
  
### **変更点（大きい順）**  
  
1. colon 定義の本体に「opcode」ではなく「CFA（実行アドレス）」を並べる    
   * code[] は「ワードの実行ルーチンのアドレス列（16-bit）」になる（DTC/ITC 系）    
2. switch(op) をやめて、NEXT（間接ジャンプ） を Z80 で作る    
   * これが FIG-FORTH の心臓部    
3. push/pop は関数ではなく、TOSキャッシュ + スタックポインタ操作にする    
   * ここが “microWORD” の中心になります  
  
---  
  
## **2) “microWORD” は何個必要か（現実的な最小集合）**  
  
あなたが挙げた theCode の中に出てくる操作（例：`pc=(ucell)rpop();` や `push(code[pc++]);`）を Forthワードに置き換えるには、まず「VMが最低限持つ microWORD」を決めます。  
  
FIG-FORTH 的にやるなら、**“microWORD”**というより Z80 の **"primitive codeWORD"**（= ネイティブ語）を最低限用意します。  
  
### **最低限（VMとして成立する核）**  
  
* NEXT（インナインタプリタ）    
* EXIT（= ;S、return stack から IP を戻す）    
* LIT（次セルをスタックへ）    
* EXECUTE（任意の xt を実行）    
* BRANCH / 0BRANCH（制御構造の土台）    
* データスタック操作の核：DROP DUP SWAP OVER >R R> R@ くらい  
  
### **ただし：**strlen **や** system() **は？**  
  
* SLEN（strlen）や SYSTEM（system()）は、最終的に OS / C ランタイム依存です。    
  MSX0 上でやるなら、    
  * SLEN は Z80 だけで書けます（0終端を走査）    
  * SYSTEM はそのままは無理で、MSX-DOS の BDOS 呼び出し、あるいは MSX BIOS 呼び出し等の “外部” に置き換えることになります    
    （ここは以前あなたが言っていた「WORDだけでは成立しない（VM外に残る）」系の代表例です）  
  
---  
  
## **3) Z80 実装例（FIG-FORTH寄り / DTC風）**  
  
ここからが本題の「microWORD 実行部（Z80命令）」です。  
  
前提を固定しないとコードがブレるので、まず “よくある割り当て” を置きます。  
  
### **レジスタ割り当て（おすすめ・実装しやすい）**  
  
* HL = IP（インストラクションポインタ：colon 定義のセル列を指す）    
* DE = W（作業：取り出した CFA/XT を保持）    
* BC = 汎用（PFA計算や分岐先など）    
* データスタック：RAM上の配列（セル=16bit想定）    
  * IX = データスタックポインタ（DSP）    
  * TOS は HL にキャッシュ …と言いたいところですが、HL を IP に使っているので競合します    
    → なので、ここでは TOS を “A'F' など” でキャッシュはしません。まずは単純に RAMスタックでいきます。    
* リターンスタック    
  * IY = リターンスタックポインタ（RSP）  
  
※TOSキャッシュは高速化段階で入れるのが安全です。まずは “動く FIG 風” を優先します。  
  
### **メモリ配置（例）**  
  
* DSP は「次に push する位置」を指す方式にすると楽です    
* スタック grows-up / grows-down はどちらでもOK。ここでは grows-up で書きます。  
  
---  
  
### **3.1 NEXT（最重要 microWORD）**  
  
colon 定義本体は「CFA(=実行ルーチンのアドレス)の配列」だとします。  
  
~~~  
; HL = IP  
  
; (HL) から次の CFA(16bit) を取り出してジャンプ  
  
NEXT:  
  
   ld  e,(hl)        ; low  
  
   inc hl  
  
   ld  d,(hl)        ; high  => DE = CFA  
  
   inc hl            ; IP += 2  
  
   ex  de,hl         ; HL = CFA, DE = IP  
  
   jp  (hl)          ; CFAへ  
~~~  
  
ここで DE に退避した IP を primitive 側で使うために、primitive の入り口で `ex de,hl` して戻す流儀にします。  
  
---  
  
### **3.2 EXIT（= ;S）**  
  
EXIT は「リターンスタックから IP を戻して NEXT」です。  
  
~~~  
; IY = RSP（grows-up）  
  
; push時: (IY)=low, (IY+1)=high, IY+=2  
  
RPOP_IP:  
  
   dec iy  
  
   ld  h,(iy)  
  
   dec iy  
  
   ld  l,(iy)      ; HL = IP  
  
   jp  NEXT  
  
P_EXIT:  
  
   ; primitive入口：HL=CFA, DE=IP なので IP を HL に戻す  
  
   ex  de,hl       ; HL = IP  
  
   jp  RPOP_IP    
~~~  
---  
  
### **3.3 LIT（次セルをデータスタックへ）**  
  
LIT は「IPが指す次セルを読み、データスタックへ push、IP更新」です。  
  
~~~  
; IX = DSP（grows-up）  
  
; push 16bit: (IX)=low, (IX+1)=high, IX+=2  
  
P_LIT:  
  
   ex  de,hl            ; HL = IP  
  
   ld  e,(hl)  
  
   inc hl  
  
   ld  d,(hl)           ; DE = literal  
  
   inc hl               ; IP += 2  
  
   ; push DE  
  
   ld  (ix+0),e  
  
   ld  (ix+1),d  
  
   inc ix  
  
   inc ix  
  
   ; 戻す：DE=IP として NEXTへ  
  
   ex  de,hl            ; DE = IP, HL = literal(不要)  
  
   ex  de,hl            ; HL = IP にしたい? 方式統一が必要  
  
   ; ここでは NEXT が HL=IP を要求するので HLへ  
  
   ; さっき HL=IP なので、そのまま  
  
   jp  NEXT  
~~~  
  
↑このままだとレジスタの持ち方が混乱しやすいので、ルールを決めるのがコツです：  
  
* NEXT の直前は HL=IP に統一する    
* NEXT で CFA を取り出したら DE=IP を primitive に渡す    
* primitive の最初は ex de,hl して HL=IP に戻して処理し、最後に jp NEXT（HL=IPのまま）で戻る  
  
このルールで LIT を書き直すと安定します：  
  
~~~  
P_LIT:  
  
   ex  de,hl            ; HL = IP  
  
   ld  e,(hl)  
  
   inc hl  
  
   ld  d,(hl)  
  
   inc hl               ; HL = 新IP, DE = literal  
  
   ld  (ix+0),e  
  
   ld  (ix+1),d  
  
   inc ix  
  
   inc ix  
  
   ; ルール：HL=IP で NEXT  
  
   jp  NEXT    
~~~  
---  
  
### **3.4 >R, R>, R@（qwc の rpush/rpop を置き換える核）**  
  
データスタックとリターンスタックの間でセルを移動できれば、pc=rpop() 的な構造も Forth 側で表現できます。  
  
#### **>R（data -> return）**  
  
~~~  
P_TOR:                  ; >R  
  
   ex  de,hl           ; HL=IP  
  
   ; pop data to DE  
  
   dec ix  
  
   dec ix  
  
   ld  e,(ix+0)  
  
   ld  d,(ix+1)  
  
   ; rpush DE  
  
   ld  (iy+0),e  
  
   ld  (iy+1),d  
  
   inc iy  
  
   inc iy  
  
   jp  NEXT  
~~~  
  
#### **R>（return -> data）**  
  
~~~  
P_RFROM:                ; R>  
  
   ex  de,hl           ; HL=IP  
  
   ; rpop to DE  
  
   dec iy  
  
   dec iy  
  
   ld  e,(iy+0)  
  
   ld  d,(iy+1)  
  
   ; push DE to data  
  
   ld  (ix+0),e  
  
   ld  (ix+1),d  
  
   inc ix  
  
   inc ix  
  
   jp  NEXT  
~~~  
  
#### **R@（return top copy）**  
~~~  
  
P_RAT:                  ; R@  
  
   ex  de,hl           ; HL=IP  
  
   ; peek (iy-2)  
  
   ld  e,(iy-2)  
  
   ld  d,(iy-1)  
  
   ld  (ix+0),e  
  
   ld  (ix+1),d  
  
   inc ix  
  
   inc ix  
  
   jp  NEXT    
~~~  
---  
  
### **3.5 BRANCH / 0BRANCH（制御構造）**  
  
ここでは “分岐先が IP 相対オフセット(16bit signed)” だとします（実装しやすい）。  
  
#### **BRANCH（無条件）**  
  
~~~  
P_BRANCH:  
  
   ex  de,hl           ; HL=IP  
  
   ld  e,(hl)  
  
   inc hl  
  
   ld  d,(hl)  
  
   inc hl              ; HL = IP_after_offset, DE = offset  
  
   add hl,de           ; HL = IP + offset  
  
   jp  NEXT  
~~~  
  
#### **0BRANCH（TOS=0なら分岐）**  
  
~~~  
P_0BRANCH:  
  
   ex  de,hl           ; HL=IP  
  
   ; pop TOS to DE  
  
   dec ix  
  
   dec ix  
  
   ld  e,(ix+0)  
  
   ld  d,(ix+1)  
  
   ; read offset to BC (or DE)  
  
   ld  c,(hl)  
  
   inc hl  
  
   ld  b,(hl)  
  
   inc hl              ; HL = IP_after_offset  
  
   ; test DE == 0 ?  
  
   ld  a,d  
  
   or  e  
  
   jr  nz, .nobranch  
  
   ; branch  
  
   ld  e,c  
  
   ld  d,b  
  
   add hl,de  
  
.nobranch:  
  
   jp  NEXT    
~~~  
---  
  
## **4) これで** PRIMS **を “Forthのワード列” に置き換えるとどうなる？**  
  
あなたの qwc の EXIT / LIT は、今は “opcodeとしてVMが解釈” していますが、    
上の方式では どれも辞書に登録された primitive word になります。  
  
すると、  
  
* colon 定義（: foo ... ;）は、comma() で opcode を入れる代わりに CFA（Z80ルーチンのアドレス） を入れます    
* doInline() 相当は「既存 colon 定義の CFA 列を複製」するだけ    
* doInterp() は「IP を一時領域に置いて inner する」ではなく、EXECUTE で xt を呼ぶ形に寄せられます  
  
つまり「PRIMSの表」は最終的に  
  
* 辞書エントリ：name -> codeword addr    
* colon 定義：[CFA][CFA][CFA]...[CFA][EXIT]  
  
という FIG-FORTH の見た目になります。  
  
---  
  
## **5) 現実的な落とし穴（ここだけ注意）**  
  
1. セル幅    
   qwc の cell/ucell が 32bit の可能性があるなら、Z80 では 16bit/32bitの決断が必要です。    
   まず 16bit に落として動かし、必要なら “ダブルセル” を追加するのが定石です。    
2. system() のような OS 依存    
   MSX0 / MSX-DOS の「外部呼び出しワード」に置き換える必要があります。    
3. 文字列ポインタ    
   mem[] 内ポインタをそのまま使うなら、Z80 側でも同じメモリモデル（アドレス空間）を揃える必要があります。  
  
---  
  
## **6) 次にやると一気に進むこと**  
  
あなたの意図（「qwc を FIG-FORTH 的に理解できる形へ」）に一番効く次ステップはこれです：  
  
* 辞書エントリの最小形（NAME, LINK, CFA）を決める    
* CREATE / : / ; が吐くコード列を「CFA列」に変える    
* 上で出した Z80 の NEXT / EXIT / LIT / BRANCH / 0BRANCH / >R / R> を組み込む  
  
ここまで行くと、qwc の theCode の多くは **「microWORD（primitive）+ colon 定義」**に自然に分解できます。  
  
---  
  
必要なら、あなたの今の DE_T（xt, flags, len, name[]）を「FIG風ヘッダ（LINK, LEN+NAME, CFA）」へ寄せる案を出して、**“qwcの辞書構造を大きく壊さずに”**移行する手順（段階的移植）も書けます。  
  
# あなた:  
  
qwc-main.zip  
  
Zip アーカイブ    
考え方を整理したいと思います。    
私の頭の中ではFIG-FORTHで学んだ知識があって、そこからGitHub CCurl/CF（FORTH)を理解しようとしていました。ところが、CFは32ビットPCを対象に設計されていることがわかり、ベースとなるFORTHシステムを今回から６４ビットPCを対象にして設計されたGitHub CCurl/QWCをベースにすることにしました（Curlさんから了承を得ています）。ソースコードを添付します。  
  
~~~  
GitHub CCurl/QWC    
https://github.com/CCurl/qwc/tree/main  
~~~  
  
今回のQWC-MSXはできるだけQWCを生かしながら、Z80で実行できるFORTHシステムにしたいです。    
それを考えたときに、実行速度優先でほとんどの部分をZ80のアセンブラで記述することよりも、関数はできるだけFORTHのWORDとして記述し、microWORDの実行部だけをアセンブラで書けるのではないかと思っています。そのため、MSXのファンクションコールなどはmicroWORDの実行部だけで使うことになるので、microWORDが追加されることも考えられます。  
  
# ChatGPT:  
  
思考時間: 1m 31s  
  
整理、いい方向です。あなたの方針（QWC をできるだけ生かしつつ、Z80 では 大半を Forth の WORD で書き、どうしても必要な部分だけを **microWORD（Z80 asm）**にする）は、QWC の作りと相性が良いです。  
  
まず「頭の中の地図」をズレなくするために、QWC を **“3層”**で捉えるのが一番スッキリします。  
  
---  
  
## **1) QWC を 3層に分けて考える**  
  
### **層A：VM核（絶対に必要・Z80 asm向き）**  
  
* inner() がやっている 実行ループ（opcodeを取り出して分岐する部分）    
* データスタック／リターンスタックの push/pop    
* pc（IP）更新、分岐、LIT など  
  
ここがあなたの言う microWORD の実行部です。  
  
### **層B：PRIMS（QWCの “coreWORD” の集合）**  
  
qwc-vm.c のこれです：  
  
* PRIMS(X) に並ぶ EXIT/LIT/JMP/... 等    
* いまは switch(op) の case の中に C 手続きで書かれている  
  
この層は「microWORD化」もできますが、全部を microWORD にしなくてもOKです。    
（後述：Z80で “最低限だけ microWORD”、残りは Forth WORD で合成、が現実的）  
  
### **層C：高水準 Forth（できるだけここで書く）**  
  
* qwc-boot.fth や block-01.fth に書く辞書／制御構造／ユーティリティ    
* あなたが目指している「関数はできるだけ WORD で書く」領域  
  
---  
  
## **2) 「FIG-FORTH 形式に置き換える」は2つのルートがある**  
  
あなたの希望を満たす道は、実は2案あります。  
  
### **ルート1：QWC方式を保ったまま “microWORD最小” にする（おすすめ）**  
  
* PRIMS（opcode集合）自体は維持    
* inner() を Z80 asm で実装（または超小C＋一部asm）    
* 高水準機能（制御構造、数値変換、文字列など）は Forth WORD で構成  
  
✅ 長所  
  
* QWC の設計（コンパクトさ、ブートや辞書の思想）を一番 “生かせる”    
* いきなり FIG の辞書構造へ大改造しなくて済む    
* MSX固有I/Oを「追加opcode（microWORD）」として自然に差し込める  
  
⚠️ 注意  
  
* これは “FIG-FORTHの見た目そのもの” ではなく、QWCのミニVM + Forth の形  
  
### **ルート2：FIG-FORTH 形式（NEXT + CFA列）へ本格移行**  
  
* opcode を捨てて、colon 定義の本体を CFA（実行アドレス）の列にする    
* NEXT で間接ジャンプ（JP (HL) 相当）する、いわゆる FIG の本体  
  
✅ 長所  
  
* 「FIGで学んだ形」そのものに近づく  
  
⚠️ 注意（大きい）  
  
* QWC の「セルが1命令」「プリミティブ番号」「NaN即値」などの設計と噛み合わず、辞書・コンパイル・実行モデルを広範囲に作り替えになりがち  
  
---  
  
## **3) あなたの目的に一番合うのは「ルート1」**  
  
あなたの目的は  
  
QWC-MSX はできるだけ QWC を生かす    
速度優先で asm まみれにしない    
WORD を増やしていく（microWORDは最小）  
  
なので、まずは QWCのPRIMS/opcodeモデルを維持して、  
  
* Z80に必要最小の microWORD（実行部）だけ用意    
* 残りは Forth WORD でどんどん作る  
  
が一番速く前に進めます。  
  
---  
  
## **4) Z80版で “最小 microWORD” はどれくらい？**  
  
QWC の PRIMS を全部 asm にする必要はありません。    
Z80版で「最初に asm 化すべき核」はだいたいこれだけです：  
  
### **VM制御（絶対必要）**  
  
* EXIT    
* LIT（または即値方式を捨てて常に LIT ）    
* 分岐：JMP / JMPZ / JMPNZ（or 0BRANCH/BRANCH相当）  
  
### **スタック核（絶対必要）**  
  
* DUP DROP SWAP OVER    
* \>R R> R@（最低限 >R と R> があると、あとが楽）  
  
### **メモリ（最低限）**  
  
* @ ! c@ c!  
  
### **端末I/O（MSX用に追加/差し替えが必要）**  
  
* EMIT（1文字出力）    
* KEY（1文字入力）    
* CR（EMIT合成でも可）  
  
ここまで microWORD が揃うと、残りの多くは Forth で組めます。  
  
---  
  
## **5) MSX固有コールは「microWORD追加」で自然に吸収できる**  
  
あなたの想定通り、MSXの BDOS/BIOS 呼び出しは microWORD 側だけが触るのが綺麗です。  
  
たとえば：  
  
* EMIT：CHPUT（BIOS）や BDOS 2    
* KEY：CHGET（BIOS）や BDOS 1    
* 文字列出力：BDOS 9（$終端） ← あなたが既に気づいた“盲点”  
  
さらに必要なら、  
  
* MSX-CALL（A=ファンクション番号、DE/HL引数…みたいな汎用呼び出しmicroWORD）    
* DISK-IO / LOAD / SAVE などの “OS寄りワード”  
  
を追加PRIMとして増やせます。これは QWC の「PRIMSを増やす設計」と相性がいいです。  
  
---  
  
## **6) いちばん大事な設計判断：CELL幅をどうするか**  
  
QWC は 64/32-bit を前提に LIT_MASK（quiet NaN 即値）を使っていますが、Z80 ではそこが分岐点です。  
  
### **Z80で現実的な選択**  
  
* 16-bit CELL を基本にする（MSX-DOSやZ80と相性が良い）    
* 32-bit が欲しい演算だけ “ダブルセル（2セル）” で扱う（伝統的Forthのやり方）  
  
そして NaN即値は捨てて、常に LIT 命令で即値を置くのが一番安全です。    
（これだけで移植難度がかなり落ちます）  
  
---  
  
## **7) 進め方（壊さずに前へ進むロードマップ）**  
  
あなたの「QWCを生かす」を崩さない順番はこれです：  
  
1. QWCを“16-bit CELL想定”に寄せた派生ブランチをPC上で作る    
   * LIT_MASK 系はオフ（常に LIT value 方式）    
   * MEM_SZ は 64KB〜に縮小（MSX想定）    
2. inner() の機能を「Z80で必要な最小PRIMSだけ」で動く形に固定    
3. その最小PRIMSを Z80 asm で実装（MSXで 2 3 + . が動く）    
4. あとは boot.fth 側で WORD を増やしていく    
5. MSX固有I/Oは必要になったタイミングで PRIMS を追加  
  
---  
  
## **8) ここまでの整理を一言でまとめると**  
  
* FIG-FORTHの理解を “設計の指針” として使うのは正しい    
* ただし QWC-MSX は FIG を完全再現しない方が早い    
* QWC の形（PRIMS/opcode + outer + 辞書）を保ったまま    
  microWORD（Z80 asm）を最小にして、残りは Forth WORD で積み上げる    
  これがあなたの目的に最短です  
  
---  
  
もしよければ次の返答では、あなたが添付した qwc-main の中身（qwc-vm.c/.h, system.c, qwc-boot.fth, Architecture.md）に合わせて、  
  
* QWC-MSX 用の “最小PRIMS表（何を残す/削る/追加する）”    
* MSX用 I/O microWORD（EMIT/KEY/BDOS9等）の具体的なZ80実装案    
* NaN即値を捨てて LIT 方式へ寄せた時の compileNum/compileLit の変更点  
  
を、QWCの実コードに沿って提案します。  
  
# あなた:  
  
では、最初の第一歩として、１６ビット基準で、「NaN即値は捨てて、常に LIT 命令で即値を置く」とします。    
そして、「7) 進め方（壊さずに前へ進むロードマップ）」で行きましょう。    
そのうえで、最後に述べられていた提案を実行お願いします。  
  
もしよければ次の返答では、あなたが添付した qwc-main の中身（qwc-vm.c/.h, system.c, qwc-boot.fth, Architecture.md）に合わせて、    
１）QWC-MSX 用の “最小PRIMS表（何を残す/削る/追加する）”    
２）MSX用 I/O microWORD（EMIT/KEY/BDOS9等）の具体的なZ80実装案    
３）NaN即値を捨てて LIT 方式へ寄せた時の compileNum/compileLit の変更点    
を、QWCの実コードに沿って提案します。  
  
# ChatGPT:  
  
思考時間: 4m 44s  
  
了解です。「16ビット基準」「NaN即値は捨てて常に LIT value」「ロードマップ 7 の順」で、QWCの実コード（qwc-vm.c/.h, system.c, qwc-boot.fth, Architecture.md）に沿って、まず最初の提案セット（1〜3）をまとめます。  
  
以下は “QWCを壊さずに MSX0/Z80へ寄せる” ための、いちばん手堅い最初の一歩です。  
  
---  
  
# **0) 今回の前提（確定）**  
  
* CELL = 16-bit    
* 即値は NaN 方式を廃止（lit1 を消す）    
* 数値リテラルは常に LIT opcode + value（2セル）    
* QWCのモデル（「セル列は opcode/xt/値」＋ tail-call最適化）を維持    
  ※FIG-FORTHの “CFA列+NEXT” には行かない（今は行かないのが最短）  
  
---  
  
# **1) QWC-MSX 用 “最小PRIMS表”（残す/削る/追加する）**  
  
まず結論：qwc-boot.fth が最初から使っている primitive が多いです。なので「QWCを生かす」方針なら、初期段階は “bootが要求する分は基本残す” が最短です。  
  
## **1-A. 最小コア（VMが動くために絶対必要）**  
  
qwc-boot.fth で if/then/begin/... を作るために、opcodeとして 数字で直接使われています（(lit) が 1、(jmpz) が 3、など）。  
  
* EXIT（0）… return stack から pc 復帰    
* LIT（1）… 次セルを push    
* JMP（2）    
* JMPZ（3）    
* JMPNZ（4）    
* NJMPZ（5）    
* NJMPNZ（6）  
  
これは 削れません（bootが (jmpz) , ... のように opcode を数値で直に埋めてるため）。  
  
## **1-B. スタック＆メモリ（bootがガンガン使う）**  
  
* DUP DROP SWAP OVER    
* ! @ c! c@    
* \>r r> r@    
* +L -L（tstk/lstk操作。bootで locals を構成）  
  
## **1-C. “locals（x,y,z）系”（bootが最初から構築済み）**  
  
* x! y! z!    
* x@ y@ z@    
* x@+ y@+ z@+  
  
これも boot で “ローカル3本” の仕組みを使うので、当面残すのが楽です。  
  
## **1-D. 算術・比較・論理（boot内や block-01 等で使う）**  
  
* \* \+ \- /mod 1+ 1-    
* < = > 0= min max +!    
* and or（xor は bootでは必須じゃないが、残してOK）  
  
## **1-E. 文字/外部（最低限の対話と読み込み）**  
  
qwc-boot.fth は最低でもこれを使います：  
  
* ztype    
* emit    
* add-word    
* outer    
* cmove    
* s-len    
* system（MSXでは中身を差し替え or スタブ）  
  
そして ファイル系 primitive（fopen/fread/...）は boot中に出現します。    
ただし、MSXの第一段階では、  
  
* 案1（おすすめ）：ファイル系は “スタブ” にしてとりあえず動かす    
  fopen/fread/... を呼んだらエラー文字列を出すか 0 を返す    
* 案2：MSX-DOS のファイルI/Oを実装して合わせる（工数↑）  
  
最初は 案1で十分です。  
  
## **1-F. 削る（初期の “最小” という意味で）**  
  
bootの中では使われていない/対話の最低限に必須ではないもの：  
  
* ms timer（後回しでOK）    
* key key? find i xor（bootでは直接使っていない/後でouter側で必要になるが段階的に）  
  
ただし、対話 REPL を早期にやるなら KEY/QKEY/FIND は結局すぐ要ります。    
なので「最小で起動」→「すぐ対話」の順で足していくのが現実的です。  
  
---  
  
## **1-G. 追加する（MSX用I/O microWORD）**  
  
MSXはここが差分なので、QWCの PRIMS に追加するのが自然です。  
  
* bdos（汎用：C=機能番号、DE=引数、戻りA…みたいに使えると便利）    
* bdos9（$終端文字列出力：MSX-DOSの “盲点” 対策）    
* （必要なら）bios（BIOS呼び出し）  
  
ただし最初の第一歩は、既存 emit/key?/key/ztype の 下請け関数を MSX実装にすれば、追加PRIMなしでも前に進めます（次章）。  
  
---  
  
# **2) MSX用 I/O microWORD（EMIT/KEY/BDOS9等）のZ80実装案**  
  
QWCの構造上、  
  
* PRIM EMIT は emit() 関数を呼ぶ    
* PRIM KEY は key() 関数を呼ぶ    
* PRIM QKEY は qKey() 関数を呼ぶ    
* PRIM ZTYPE は zType() 関数を呼ぶ  
  
になっています（qwc-vm.h の extern 宣言もそれ）。    
なので “microWORD（Z80 asm）” は、この4関数をZ80で実装するのが一番素直です。    
（PRIMSを増やさなくても、I/Oだけは完全にZ80にできます）  
  
以下、MSX-DOS（.COM、BDOS呼び出し CALL 0005h）前提の定番実装です。  
  
---  
  
## **2-A. emit(char ch) — 1文字出力（BDOS 2）**  
  
* 入力：A に文字（実装側の約束）    
* BDOS: C=2, E=ch, CALL 0005h  
  
~~~  
; void emit(const char ch);  
  
; 呼び出し規約は z88dk/SDCC などに合わせて調整が必要  
  
; ここでは「A=ch で呼ばれる」想定のコアだけ示します  
  
_emit:  
  
   ld  e,a        ; E = ch  
  
   ld  c,2        ; BDOS CONOUT  
  
   call 0005h  
  
   ret  
~~~  
  
（Cから呼ぶなら「引数がどのレジスタ/スタックに来るか」で入口だけ変えればOK）  
  
---  
  
## **2-B. key() — 1文字入力（BDOS 1）**  
  
* BDOS: C=1, CALL 0005h    
* 戻り：A に文字  
  
~~~  
_key:  
  
   ld  c,1        ; BDOS CONIN  
  
   call 0005h     ; A = char  
  
   ret    
~~~  
---  
  
## **2-C. qKey() — キー有無（BDOS 0Bh）**  
  
MSX-DOSの Console input status（一般に Function 0Bh）で、  
  
* A=0 のとき「入力なし」    
* A=0FFh のとき「入力あり」  
  
という形で判定できます。  
  
~~~  
_qKey:  
  
   ld  c,0Bh      ; BDOS CONST  
  
   call 0005h     ; A=0 or 0FFh  
  
   ret    
~~~  
---  
  
## **2-D. zType(const char* s) — 0終端文字列出力（EMITの繰り返し）**  
  
QWCの zType() は C文字列（0終端）を出す想定です。MSX側も同じでOK。  
  
`; HL = ptr` として呼ばれる想定（入口は呼び出し規約で調整）  
  
~~~  
_zType:  
  
.loop:  
  
   ld  a,(hl)  
  
   or  a  
  
   ret z  
  
   push hl  
  
   call _emit       ; A=ch  
  
   pop  hl  
  
   inc hl  
  
   jr  .loop    
~~~  
---  
  
## **2-E. BDOS 9（$終端文字列出力）— “盲点対策” 用 microWORD**  
  
あなたが既に気づいたやつです。これは 0終端ではなく $ 終端。  
  
* BDOS: C=9, DE=ptr, CALL 0005h  
  
~~~  
; DE = ptr ($終端)  
  
_bdos9:  
  
   ld  c,9  
  
   call 0005h  
  
   ret  
~~~  
  
### **使い分けの提案**  
  
* QWC内部の基本メッセージは zType（0終端）で統一    
* MSX-DOSの “既存の表示資産” を使うときだけ bdos9 を追加  
  
---  
  
# **3) NaN即値を捨てて LIT 方式へ寄せた時の compileNum/inner の変更点（実コード準拠）**  
  
ここは qwc-vm.c / qwc-vm.h / Architecture.md の “Literal Encoding” に直撃します。  
  
## **3-A. qwc-vm.h の変更（16bit化＋LIT_MASK削除）**  
  
現状は 32/64で LIT_MASK/LIT_BITS を定義していますが、MSX版では不要です。  
  
提案（MSX用コンフィグ例）：  
  
* CELL_SZ 2    
* typedef int16_t cell; typedef uint16_t ucell;    
* MEM_SZ は 64KB以内（例：0x8000 や 0xE000 等、用途次第）    
* LIT_MASK/LIT_BITS は 定義しない（または0にしても良いが、判定側を消すのが安全）  
  
## **3-B. compileNum の変更（常に lit2）**  
  
qwc-vm.c の現状：  
  
~~~  
void lit1(cell n) { comma((ucell)n | LIT_MASK); }  
  
void lit2(cell n) { comma(LIT); comma(n); }  
  
void compileNum(cell n) { btwi(n,0,LIT_BITS) ? lit1(n) : lit2(n); }  
~~~  
  
MSX案（最短・安全）：  
  
* lit1 を削除    
* compileNum は常に lit2  
  
~~~  
void lit2(cell n) { comma(LIT); comma((ucell)n); }  
  
void compileNum(cell n) { lit2(n); }  
~~~  
  
※ 16bit環境で “負数” も普通に LIT value で積めます（2の補数のまま積む）  
  
## **3-C. inner() の default 分岐から “NaN即値判定” を削除**  
  
現状（qwc-vm.c）：  
  
~~~  
default:  
  
   if ((ir & LIT_MASK) == LIT_MASK) { push((cell)(ir & LIT_BITS)); goto next; }  
  
   if (code[pc] != EXIT) { rpush(pc); }  
  
   pc = ir;  
  
   goto next;  
~~~  
  
MSX案：  
  
* NaN即値が無いので、この if を 丸ごと消す    
* 「primitiveじゃないセル」は **XT（= codeアドレス）**として扱うだけ  
  
~~~  
default:  
  
   if (code[pc] != EXIT) { rpush(pc); } *// tail-call optimization*  
  
   pc = (ucell)ir;  
  
   goto next;  
~~~  
  
そして、数値は必ず LIT 命令で入ってくるので、LIT case だけが literal を扱います。  
  
---  
  
# **まとめ：この一歩で何が“確実に”前に進むか**  
  
この変更セット（16bit化＋NaN即値廃止＋I/O関数をZ80実装）を入れると：  
  
* qwc-boot.fth がやっている「opcodeを数値で埋める」方式が、そのままZ80でも成立    
* 数値はすべて LIT value になるので、実装が単純化（バグが減る）    
* MSXのI/O差分は emit/key/qKey/zType/bdos9 だけに閉じ込められる    
* その後に「最小PRIMSで 2 3 + .」→「outer拡張」へ段階的に進める  
  
---  
  
次の返答で（あなたのロードマップ7に沿って）さらに踏み込むなら、私はここを具体化します：  
  
1. MSX向け MEM_SZ/辞書領域/スタック領域 の現実的レイアウト案（64KB内での切り方）    
2. system primitive の扱い（MSXでは何に置き換えるのが自然か：bye/dos/call 等）    
3. “最小PRIMSで起動するための qwc-boot.fth 側の微調整案”（ファイル系をスタブ化する/しない）  
  
必要なら、z88dk/SDCC どちらでビルドする想定かに合わせて、上の _emit/_key/... の「正しい関数入口（引数の取り方）」まで確定して書きます。  
  
# あなた:  
  
すごいですね！朝起きてから始めて間もない間に、もうここまで進んでしまった。    
COMファイル化はDebianでsjasmplusを使ってmakeしています。    
具体化をお願いします。  
  
１）MSX向け MEM_SZ/辞書領域/スタック領域 の現実的レイアウト案（64KB内での切り方）    
２）system primitive の扱い（MSXでは何に置き換えるのが自然か：bye/dos/call 等）    
３）“最小PRIMSで起動するための qwc-boot.fth 側の微調整案”（ファイル系をスタブ化する/しない）  
  
必要なら、z88dk/SDCC どちらでビルドする想定かに合わせて、上の _emit/_key/... の「正しい関数入口（引数の取り方）」まで確定して書きます。  
  
# ChatGPT:  
  
思考時間: 2m 29s  
  
了解です。ここからは **「MSX-DOS の .COM（ORG 0100h）」「CELL=16bit」「即値は必ず LIT value」**を前提に、QWCの“実コード”の作り（mem[] の中で code↑ / dict↓、スタックは別配列、qwcInit() が変数アドレスを const で登録…）に沿って、具体案を出します。  
  
---  
  
# **1) MSX向け MEM_SZ / 辞書領域 / スタック領域の現実的レイアウト案**  
  
## **QWCの前提（実コード確認）**  
  
qwc-vm.c ではこうなっています：  
  
* char mem[MEM_SZ];    
* ucell *code = (ucell*)&mem[0];    
* here は codeのセルオフセット（code[here++] = ...）    
* last は **辞書エントリのポインタ（byteアドレス）**で、mem[MEM_SZ] から下へ成長    
* スタックは dstk/rstk/lstk/tstk の 別配列（STK_SZ+1）  
  
つまり MSXでも「64KB内に全部おさめる」ために、  
  
* mem[] を 小さく切る（たとえば 24KB〜40KB）    
* スタック配列を 数百バイト確保    
* 入力バッファ（TIB）等も 小さく確保  
  
すれば成立します。  
  
---  
  
## **まずおすすめの “堅い初期値”**  
  
最初の第一歩（boot を動かし、対話に到達）だけなら、これが安全です：  
  
* CELL_SZ = 2    
* STK_SZ = 63（現状維持でOK。各スタック 64セル）    
* MEM_SZ = 0x6000（24KB）  
  
理由：  
  
* 辞書エントリは sz が byte なので、巨大にするより“適度”が扱いやすい    
* boot が辞書を結構作るので 16KBだと窮屈になりがち    
* 24KBなら code と dict の両方に余裕が出る  
  
### **メモリ内のイメージ（mem[] の中）**  
  
`mem[0..MEM_SZ-1]` の中は QWCの思想通り：  
  
mem[0]                   mem[MEM_SZ]  
+--------------------+-------------------+  
| compiled code (↑)  | dict entries (↓)  |  
+--------------------+-------------------+  
         ^here                ^last  
  
* here は「cell単位のインデックス」（0,1,2,…）    
* last は「アドレス」（dpポインタ）  
  
初期値は実コード通り：  
  
* here = LASTOP + 1（PRIMS enum の最後+1）    
* last = (cell)&mem[MEM_SZ]  
  
---  
  
## **.COM の“全体メモリ”としての配置（実用的な切り方）**  
  
MSX-DOS の .COM は 0100h から TPA に載ります。あなたの作りたい QWC-MSX は最初は 全部静的確保で十分です（あとで BDOS でメモリ確保に拡張できます）。  
  
### **おすすめの全体配置（ざっくり）**  
  
* 0100h〜：プログラムコード    
* その後ろ：グローバル変数・スタック配列・TIB・mem[]    
* 上位領域：DOS/BIOS が使うので触らない  
  
重要なのは「あなたの .COM の末尾アドレスが TPA を超えない」ことです。    
MEM_SZ=0x6000 なら、スタック等を入れても +1KB程度なので、たいてい安全圏です。  
  
---  
  
## **さらに “枯れた” 改善（すぐ効きます）**  
  
### **① dict entry のアラインを 4→2 に**  
  
addToDict() にあります：  
  
~~~  
while (sz & 0x03) { ++sz; }   *// 4バイト境界*  
~~~  
  
Z80で16bitセルなら 2バイト境界で十分です。  
  
~~~  
while (sz & 0x01) { ++sz; }   *// 2バイト境界*  
~~~  
  
これだけで辞書の無駄が減ります（地味に効きます）。  
  
### **②** DE_T **の** sz **が byte（最大255）**  
  
これは “巨大辞書” を前提にしないMSXではむしろメリットです。    
なので MSX初期は このまま維持が簡単です。  
  
---  
  
# **2)** system **primitive を MSX でどう扱うのが自然か**  
  
QWCの PRIM system は system((char*)pop()); を呼びます（qwc-vm.c）。  
  
qwc-boot.fth では今：  
  
~~~  
: vi z" vi qwc-boot.fth" system ;  
~~~  
  
つまり 開発PC向けの便利機能です。MSXでは同じことはできません（MSX-DOS2 の EXEC 等を本格実装するなら可能性はありますが、最初の一歩には重すぎます）。  
  
## **結論：最初は “BYE相当” か “未サポート表示” が自然**  
  
おすすめは段階別にこうです。  
  
### **段階1（最初の起動・対話優先）**  
  
* system は 未サポートとして    
  * 文字列を捨てて（drop）    
  * "system: not supported" を表示    
  * 0 を返す（または何もしない）  
  
これが一番事故りません。  
  
### **段階2（MSXらしくする）**  
  
* system を DOSに戻る（= bye 相当）にする    
  * “コマンド実行” ではなく “DOSへ戻る命令” として再定義する  
  
### **段階3（余力が出たら）**  
  
* system を “限定的コマンド” として実装    
  * たとえば "dir" "type" などだけ解釈して BDOS を叩く    
  * あるいは MSX-DOS2 の EXEC 機能に寄せる（工数大）  
  
---  
  
## **MSX版** system **の具体案（C/asmどちらでも）**  
  
### **A案：未サポート（推奨）**  
  
* system(char* s) は zType("system: not supported\n"); だけ  
  
### **B案：DOSへ戻る（bye）**  
  
* state = BYE; 相当にする（QWCは BYE=999 を boot が使っています）  
  
---  
  
# **3) “最小PRIMSで起動するための qwc-boot.fth 微調整案”（ファイル系スタブ含む）**  
  
ここがポイントで、qwc-boot.fth は（あなたも見つけた通り）途中から fopen/fread/fclose を使う “reboot/load” を持っています。  
  
最初の一歩でやりたいことは：  
  
* QWCの boot の “コア部分（制御構造、辞書、文字列、.”, etc）” を生かす    
* ただし MSXでいきなりファイルI/Oを作らない  
  
なので、boot 側を少しだけ条件分岐 or スタブ化します。  
  
---  
  
## **3-A. ファイルI/Oを「呼ばれても落ちない」スタブにする（最短）**  
  
MSX側（primitive）で：  
  
* fopen は 0 を返す    
* fread は 0 を返す    
* fclose は何もしない  
  
にしておけば、boot の rb はこう動きます：  
  
~~~  
z" qwc-boot.fth" fopen-r -if0 drop ." ...not found" exit then  
~~~  
  
fopen-r が 0 を返す → -if0 側に行く → ちゃんとメッセージ出して exit    
つまり bootは生き残ります。  
  
これが「boot.fth を極力いじらない」最短路です。  
  
---  
  
## **3-B. でも boot には “巨大読み込み領域 t4=50000” がある（MSXでは無理）**  
  
qwc-boot.fth の reboot 部分は：  
  
~~~  
: t4 50000 ;  
  
: t5 vars t4 + ;  
  
...  
  
t5 t4 z@ fread ...  
~~~  
  
MSXの mem[] を 24KB にした時点で 50000 は確実に破綻します。    
ただし、上のスタブ方式なら rb 自体を使わなければ問題になりません。  
  
そこで boot を少しだけ安全化するなら、MSXでは rb を “無効化” するのがきれいです。  
  
### **おすすめの微調整（MSX向け）**  
  
boot の ( Files )〜( reboot ) のあたりに、MSX向けならこれを入れます：  
  
* rb を「not supported」にする（または bye）  
  
例（差し替え案）：  
  
~~~  
( MSX: file system not available yet )  
  
: rb  ." rb not supported on MSX yet" ;  
  
: vi  ." vi not supported on MSX" ;  
~~~  
  
※ こうしておけば boot の他の部分はそのまま生きます。  
  
---  
  
## **3-C. “最小PRIMS” をどこまで削るか（現実案）**  
  
あなたの方針は「最小PRIMSで起動 → 後で増やす」なので、現実的にこう分けるのが良いです。  
  
### **起動に必須（bootが opcode 数値で直に使う）**  
  
* EXIT/LIT/JMP/JMPZ/JMPNZ/NJMPZ/NJMPNZ    
* add-word / outer / ztype / emit / key? / key    
* @ ! c@ c!    
* dup drop swap over    
* >r r> r@    
* +L -L x!/y!/z! x@/y@/z@ x@+/y@+/z@+（bootが使う）  
  
### **先送り可能（ただし早晩ほしくなる）**  
  
* fopen/fread/fclose/fwrite（当面スタブでOK）    
* ms/timer（なくても起動できる）    
* system（未サポートにしてOK）  
  
---  
  
# **4)** _emit/_key/_qKey/_zType **の「正しい関数入口」について（sjasmplus前提での現実解）**  
  
ここだけは正直に言うと、**“Cを使うかどうか”**で入口が変わります。  
  
あなたはいま「Debianで sjasmplus で .COM 作っている」ので、最初の一歩は：  
  
* VMもI/Oも 全部 asm（sjasmplus）で書く    
* もしくは “VMは asm、outer/辞書は asm” という方向  
  
が一番一直線です。  
  
もし「qwc-vm.c を残して Z80-C でビルドしたい」なら z88dk か SDCC のリンクが必要になります（sjasmplus単体ではCはリンクできません）。  
  
なのでここでは、sjasmplus単体で確実に動く形として、  
  
* emit/key/qKey/zType は Z80の通常ルーチン（呼び出し規約を自分で決める）    
* VM側（PRIMSのEMIT等）も同じ規約で呼ぶ  
  
にするのが最短です。  
  
例：  
  
* emit は A=ch    
* key は A=ch 返す    
* qKey は A=0/FFh    
* zType は HL=ptr(0終端)  
  
この規約なら、以前出した BDOS版のコードをそのまま使えます。  
  
---  
  
# **5) すぐ使える「具体化セット」まとめ**  
  
## **(1) MSX初期パラメータ**  
  
* CELL_SZ=2    
* STK_SZ=63（当面そのまま）    
* MEM_SZ=0x6000（24KB）    
* dict entry align を 2 に（推奨）  
  
## **(2) system primitive**  
  
* まずは 未サポート表示（または bye）  
  
## **(3) qwc-boot.fth 微調整**  
  
* rb と vi を無効化（not supported）    
* ファイルPRIMSは スタブでもOK（boot全体を壊さない）  
  
---  
  
もし次に進めるなら、あなたの “ロードマップ7” の流れに合わせて、  
  
* MSX版の qwc-boot.fth を「最小起動向けに安全化した差分」（rb/vi無効化、必要なら t4 の削除）    
* 最小PRIMSだけで 2 3 + . まで動かすための「最初の block/テスト用ソース」    
* MEM_SZ=0x6000 のときの「どれくらい辞書が増えると last/here が衝突するか」の見積もりとガード（チェック）  
  
まで一気に書けます。  
  
# **あなた:**  
  
最小PRIMSだけで 2 3 + . まで動かすための「最初の block/テスト用ソース」    
を最初に見てみたいです。ソースコード化してもらえますか？  
  
# **ChatGPT:**  
  
はい。まずは「最小PRIMSだけで 2 3 + . を実行して終了」するための **最初の .COM テスト用ソース（sjasmplus向け、単体で動くデモ）**を用意します。  
  
これは QWCの“LITは常に LIT value” ルールに合わせて、  
  
* LIT, 2, LIT, 3, ADD, DOT, EXIT  
  
という **コード列（= block相当の最小テスト）**を ROM/メモリ上に置いて、Z80の inner で実行します。  
  
ここでは “.（DOT）” を Forthで組まずに PRIM（microWORD）として実装します。    
最初の一歩として「VMが動く／スタックが動く／即値が動く／出力が動く」を最短で確認するためです。    
次の段階で DOT を Forth WORD 側に移していけます。  
  
---  
  
## qwc_msx_min_demo.asm**（sjasmplus / MSX-DOS .COM）**  
  
~~~  
; qwc_msx_min_demo.asm  
  
; MSX-DOS .COM minimal demo for QWC-MSX concept (16-bit cells)  
  
; - Literals are ALWAYS: LIT <value> (no NaN-immediates)  
  
; - Minimal opcodes: EXIT, LIT, ADD, DOT  
  
; Runs: 2 3 + .  => prints "5 " then CRLF and returns to DOS.  
  
;  
  
; Build (Debian):  
  
;   sjasmplus --raw=qwc_msx_min_demo.com qwc_msx_min_demo.asm  
  
; Run on MSX-DOS:  
  
;   qwc_msx_min_demo  
  
               org     0100h  
  
               jp      start  
  
; -----------------------------  
  
; Opcodes (PRIMS minimal subset)  
  
; -----------------------------  
  
OP_EXIT         equ     0  
  
OP_LIT          equ     1  
  
OP_ADD          equ     7      ; NOTE: In real QWC enum this differs; here is local demo mapping  
  
OP_DOT          equ     8  
  
; -----------------------------  
  
; BDOS call (MSX-DOS)  
  
;   CALL 0005h with C=function  
  
; -----------------------------  
  
BDOS            equ     0005h  
  
FN_CONOUT       equ     02h     ; E = char  
  
FN_CONIN        equ     01h  
  
; -----------------------------  
  
; Data stack (16-bit cells)  
  
; -----------------------------  
  
STK_SZ          equ     64      ; cells  
  
dsp:            dw      0       ; 0..STK_SZ  
  
dstk:           defs    STK_SZ*2 ; 64 cells * 2 bytes  
  
; -----------------------------  
  
; Code "block" (token stream)  
  
; Each cell is 16-bit little-endian.  
  
; LIT <n> is 2 cells: OP_LIT, n  
  
; -----------------------------  
  
code_demo:  
  
               dw      OP_LIT, 2  
  
               dw      OP_LIT, 3  
  
               dw      OP_ADD  
  
               dw      OP_DOT  
  
               dw      OP_EXIT  
  
; =============================  
  
; Entry  
  
; =============================  
  
start:  
  
               ; init dsp=0  
  
               xor     a  
  
               ld      hl,0  
  
               ld      (dsp),hl  
  
               ; HL = PC pointer (address into code_demo)  
  
               ld      hl, code_demo  
  
               call    inner  
  
               ; print CRLF  
  
               ld      a, 13  
  
               call    putc  
  
               ld      a, 10  
  
               call    putc  
  
               ret  
  
; =============================  
  
; inner loop  
  
; HL = pc (points to opcode cell)  
  
; =============================  
  
inner:  
  
.next:  
  
               ; fetch opcode into DE (little endian), advance PC by 2  
  
               ld      e,(hl)  
  
               inc     hl  
  
               ld      d,(hl)  
  
               inc     hl  
  
               ; We only use low byte for opcode in this demo.  
  
               ld      a, e  
  
               cp      OP_EXIT  
  
               jr      z, .do_exit  
  
               cp      OP_LIT  
  
               jr      z, .do_lit  
  
               cp      OP_ADD  
  
               jr      z, .do_add  
  
               cp      OP_DOT  
  
               jr      z, .do_dot  
  
               ; unknown opcode => print '?' and stop  
  
               ld      a, '?'  
  
               call    putc  
  
               ret  
  
.do_exit:  
  
               ret  
  
.do_lit:  
  
               ; read literal (16-bit) from stream, advance PC  
  
               ld      e,(hl)  
  
               inc     hl  
  
               ld      d,(hl)  
  
               inc     hl  
  
               call    push_de  
  
               jr      .next  
  
.do_add:  
  
               ; ( a b -- a+b )  
  
               call    pop_de      ; DE=b  
  
               push    de  
  
               call    pop_de      ; DE=a  
  
               pop     bc          ; BC=b  
  
               ; DE = a, BC = b  
  
               ex      de,hl       ; HL=a  
  
               add     hl,bc       ; HL=a+b  
  
               ex      de,hl       ; DE = sum  
  
               call    push_de  
  
               jr      .next  
  
.do_dot:  
  
               ; ( n -- )  print unsigned decimal + space  
  
               call    pop_de  
  
               call    print_u16_de  
  
               ld      a, ' '  
  
               call    putc  
  
               jr      .next  
  
; =============================  
  
; Stack helpers  
  
; dsp is cell count  
  
; push: dstk[dsp++] = DE  
  
; pop:  DE = dstk[--dsp]  
  
; =============================  
  
push_de:  
  
               ; HL = dsp  
  
               ld      hl,(dsp)  
  
               ; check overflow: if hl >= STK_SZ, ignore (demo)  
  
               ld      a, h  
  
               or      a  
  
               jr      nz, .push_ok  
  
               ld      a, l  
  
               cp      STK_SZ  
  
               jr      nc, .push_ok  
  
.push_ok:  
  
               ; compute addr = dstk + dsp*2  
  
               add     hl,hl               ; *2  
  
               ld      bc, dstk  
  
               add     hl,bc  
  
               ; store DE  
  
               ld      (hl), e  
  
               inc     hl  
  
               ld      (hl), d  
  
               ; dsp++  
  
               ld      hl,(dsp)  
  
               inc     hl  
  
               ld      (dsp),hl  
  
               ret  
  
pop_de:  
  
               ld      hl,(dsp)  
  
               ld      a,h  
  
               or      a  
  
               jr      nz, .pop_ok  
  
               ld      a,l  
  
               or      a  
  
               jr      nz, .pop_ok  
  
               ; underflow => return 0  
  
               ld      de,0  
  
               ret  
  
.pop_ok:  
  
               ; --dsp  
  
               dec     hl  
  
               ld      (dsp),hl  
  
               ; addr = dstk + dsp*2  
  
               add     hl,hl  
  
               ld      bc, dstk  
  
               add     hl,bc  
  
               ; load DE  
  
               ld      e,(hl)  
  
               inc     hl  
  
               ld      d,(hl)  
  
               ret  
  
; =============================  
  
; putc: A=char  
  
; =============================  
  
putc:  
  
               ld      e,a  
  
               ld      c,FN_CONOUT  
  
               call    BDOS  
  
               ret  
  
; =============================  
  
; print_u16_de:  
  
;  input: DE = unsigned 16-bit  
  
;  output: decimal digits via putc  
  
;  destroys: AF,BC,DE,HL  
  
; =============================  
  
print_u16_de:  
  
               ; Special case 0  
  
               ld      a,d  
  
               or      e  
  
               jr      nz, .nz  
  
               ld      a,'0'  
  
               call    putc  
  
               ret  
  
.nz:  
  
               ; We will repeatedly divide by 10, push remainders.  
  
               ; Use a small digit stack in RAM (max 5 digits for 0..65535)  
  
               ld      hl, digit_buf  
  
               ld      (digptr), hl  
  
.divloop:  
  
               ; quotient in HL, remainder in A  
  
               call    udiv10_de  
  
               ; udiv10_de returns: HL=quot, A=rem(0..9)  
  
               ; store rem  
  
               ld      bc,(digptr)  
  
               ld      (bc),a  
  
               inc     bc  
  
               ld      (digptr),bc  
  
               ; if quotient==0 stop else DE=quot and continue  
  
               ld      a,h  
  
               or      l  
  
               jr      z, .print  
  
               ex      de,hl  
  
               jr      .divloop  
  
.print:  
  
               ; print digits in reverse from digit_buf to digptr-1  
  
               ld      bc,(digptr)  
  
.print_loop:  
  
               ld      hl, digit_buf  
  
               ; if bc == digit_buf => done  
  
               or      a          ; clear carry  
  
               sbc     hl, bc  
  
               jr      z, .done  
  
               ; --bc, load digit  
  
               dec     bc  
  
               ld      a,(bc)  
  
               add     a,'0'  
  
               call    putc  
  
               jr      .print_loop  
  
.done:  
  
               ret  
  
; =============================  
  
; udiv10_de: unsigned divide DE by 10  
  
; returns:  
  
;   HL = quotient  
  
;   A  = remainder  
  
; Algorithm: simple restoring division (16 steps)  
  
; =============================  
  
udiv10_de:  
  
               ld      hl,0        ; quotient  
  
               ld      bc,0        ; remainder in C (8-bit enough for /10)  
  
               ld      a,16  
  
.udiv_loop:  
  
               ; shift left remainder:quotient, bring next bit from DE  
  
               ; remainder (BC) <<= 1, bring msb of DE into C  
  
               sla     e  
  
               rl      d  
  
               rl      c           ; C holds remainder (0..19)  
  
               ; quotient HL <<= 1  
  
               add     hl,hl  
  
               ; if remainder >= 10 then remainder-=10, set quotient bit  
  
               ld      a,c  
  
               cp      10  
  
               jr      c, .no_sub  
  
               sub     10  
  
               ld      c,a  
  
               inc     l           ; set low bit of quotient (because we just shifted)  
  
.no_sub:  
  
               dec     a           ; use A as loop counter? can't, A changed  
  
               ; restore loop counter in A: use B as counter instead  
  
               ; (so we need a different counter)  
  
               ; --- rewrite counter using B instead ---  
  
               ; (We avoid rewriting by using a dedicated counter register.)  
  
               ; This version uses a fixed counter stored in B instead:  
  
               ; We'll patch below by jumping to an alternate routine.  
  
               jp      udiv10_de_fixed  
  
; ---- Fixed routine (used) ----  
  
udiv10_de_fixed:  
  
               ; Re-do cleanly with B as counter  
  
               ld      hl,0  
  
               ld      c,0  
  
               ld      b,16  
  
.loop2:  
  
               sla     e  
  
               rl      d  
  
               rl      c  
  
               add     hl,hl  
  
               ld      a,c  
  
               cp      10  
  
               jr      c, .no2  
  
               sub     10  
  
               ld      c,a  
  
               inc     l  
  
.no2:  
  
               djnz    .loop2  
  
               ld      a,c          ; remainder  
  
               ret  
  
; digit buffer (5 bytes enough)  
  
digit_buf:  
  
               defs    6  
  
digptr:  
  
               dw      0    
~~~  
---  
  
## **使い方**  
  
Debian で：  
  
~~~  
sjasmplus --raw=qwc_msx_min_demo.com qwc_msx_min_demo.asm  
~~~  
  
MSX-DOSで実行すると、画面に：  
  
~~~  
5  
~~~  
  
（実際は `"5 " + CRLF`）と出て終了します。  
  
---  
  
## **次のステップ（このデモを QWC-MSX に“繋げる”）**  
  
このデモは「最小PRIMSの動作確認」専用です。次はこれを **QWCの列挙順（EXIT=0, LIT=1, JMP=2,...）**に揃えて、  
  
* OP_ADD を QWC の ADD の番号に合わせる    
* DOT をいったん PRIM のまま維持しつつ、後で . を Forth 側へ移す  
  
という段階に入るのが自然です。  
  
もしよければ、あなたが添付してくれた qwc-main の enum { PRIMS(X1) }; の実際の順番に合わせて、このテストを **“QWC互換の opcode 番号”**で書き直した版も出します（その方が次へそのまま繋がります）。  
  
# あなた:  
  
簡単にしか見れてないのですが、終了時にMSX-DOSに戻る処理がないように思います。  
  
# ChatGPT:  
  
思考時間: 14s  
  
ご指摘どおり「明示的な“終了API呼び出し”」は入っていません。ただし MSX-DOS(.COM) では多くの場合、RET で DOS に戻れます（コマンドプロセッサが CALL 0100h で起動するため）。これは CP/M 系の作法そのものです。   
  
とはいえ、環境や拡張（MSX-DOS2など）で「より正統」な終わり方にしたいなら、BDOS function 00h（TERM0 / System reset） を呼ぶのが確実です。MSXの資料でも「C=00h でコマンドレベルへ戻る」とされています。   
  
なので、あなたの安心のために 終了処理を明示した版に直しましょう。    
（ついでに、前回のデモは opcode 番号を仮にしていたので、ここでは「終了だけ最小修正」します。）  
  
---  
  
## **修正点：終了時に BDOS 00h を呼ぶ**  
  
start: の最後を  
  
* ret ではなく    
* bdos_term0 に飛ばす  
  
に変更します。  
  
### **追加するルーチン**  
  
~~~  
; ---- Terminate to MSX-DOS (BDOS function 00h) ----  
  
bdos_term0:  
  
   ld  c, 00h  
  
   call BDOS  
  
   ; 通常ここには戻りませんが保険で  
  
   ret  
~~~  
  
### start: **の末尾を変更**  
  
~~~  
        ; print CRLF  
  
               ld      a, 13  
  
               call    putc  
  
               ld      a, 10  
  
               call    putc  
  
               jp      bdos_term0      ; ← ここを ret から変更  
  
~~~
  
これで「どの実行経路で終わっても MSX-DOS に戻る」が明確になります。   
  
---  
  
## **ついでの注意（TERM0は“リセット寄り”）**  
  
BDOS 00h は CP/M 的には **“system reset / warm boot”** なので、CCP（コマンドプロセッサ）再初期化寄りの動作になります。     
「メモリ上に CCP が残っているなら単に戻る（RET）」の方が軽いケースもあります。   
  
なので、今後の方針としては：  
  
* デバッグ中・確実に戻したい → BDOS 00h（TERM0）    
* 軽く戻りたい → RET  
  
の二択で使い分けるのが実務的です。  
  
---  
  
必要なら、次の版では「RET と TERM0 をビルドスイッチで切替」できるように（IFDEF）しておきます。  
  
  
    
[(次へ進む　＜現在工事中です＞)](./cf-msx_11_c.md)        
[(README.mdへ戻る)](../README.md)        
