  
あなた:  
forth_cf_msx_demo.asmの次のステップに移る前にsystem.cの整理がしたいです。次の１）２）を教えてもらえますか？  
  
　１）system.c内部の関数の役割の説明リスト  
　２）main関数の内部構造と処理の進行手順  
  
  
  
◎system.c内部の関数  
  
void ttyMode(int isRaw)  
int qKey()  
int key()  
void ms(cell sleepForMS)  
  
cell timer()   
void zType(const char* str)   
void emit(const char ch)   
  
cell fOpen(cell name, cell mode)   
void fClose(cell fh) { fclose((FILE*)fh); }  
cell fRead(cell buf, cell sz, cell fh)   
cell fWrite(cell buf, cell sz, cell fh)   
cell fSeek(cell fh, cell offset)  
  
char tib[256];  
  
void repl()  
void boot(const char *fn)  
  
  
◎main関数  
int main(int argc, char *argv[]) {  
	cfInit();  
	addLit("argc", (cell)argc);  
	strcpy(tib, "argX");  
	for (int i=0; (i<argc) && (i<10); i++) {  
		tib[3] = '0' + i;  
		addLit(tib, (cell)argv[i]);  
	}  
	boot((1<argc) ? argv[1] : 0);  
	while (1) { repl(); }  
	return 0;  
}  
  
ChatGPT:  
以下、提示してくれた system.c の関数群と main() について、「何のためにあるか」「VM/REPLのどこで使われるか」が見えるように整理して説明します（forth-dwc / cf 系の“最小OS層”としての system.c という前提で書きます）。  
  
1) system.c 内部の関数の役割（説明リスト）  
端末（キーボード入出力・raw入力）系  
void ttyMode(int isRaw)  
  
端末（TTY）を raw/non-raw に切り替える。  
  
raw にすると、行バッファやエコー等を無効化して 1文字単位で読めるようになる（ゲームやREPLの即時キー入力向け）。  
  
REPL開始時に raw にして、終了時に元に戻す…のような用途が典型。  
  
int qKey()  
  
“キーが来ているか？”を ブロックせずに調べる関数（query key）。  
  
実装は select() や kbhit 相当の仕組みを使うことが多い。  
  
ms() の待ちループや、ノンブロッキング入力処理で使われがち。  
  
int key()  
  
1文字読む（通常は ブロックする）。  
  
REPL の入力取り込み、あるいは Forth の key プリミティブに直結する入口。  
  
void emit(const char ch)  
  
1文字出力する（Forthの emit 相当）。  
  
端末表示や zType() の内部でも使える。  
  
void zType(const char* str)  
  
文字列を出力（Forthの type 相当の“C側ユーティリティ”）。  
  
実装としては while(*str) emit(*str++); みたいな形になることが多い。  
  
起動メッセージ、エラー表示、REPLプロンプトなどで使う。  
  
時間（スリープ・タイマ）系  
void ms(cell sleepForMS)  
  
指定ミリ秒スリープ（Forthの ms 相当）。  
  
実装は usleep/nanosleep/WindowsならSleepなど。  
  
cell timer()  
  
“現在時刻”または“起動後の経過時間”を返す（単位は実装依存）。  
  
Forth側で timer を使って経過時間を測ったり、乱数種に使ったりする。  
  
ファイルI/O（Forthのファイルプリミティブ）系  
cell fOpen(cell name, cell mode)  
  
ファイルを開く（戻り値は FILE* を cell に詰めたハンドルであることが多い）。  
  
name はC文字列ポインタ（Forth文字列→C文字列変換後）を想定してることが多い。  
  
mode は "r" "w" "a" "rb" 等のポインタ、またはモード番号→内部で文字列化、など実装次第。  
  
void fClose(cell fh)  
  
FILE* を閉じる（提示の通り fclose((FILE*)fh)）。  
  
cell fRead(cell buf, cell sz, cell fh)  
  
fh から sz バイト読み、buf に入れる。  
  
返り値は実際に読めたサイズ（fread の戻り）にするのが一般的。  
  
cell fWrite(cell buf, cell sz, cell fh)  
  
buf から sz バイト書き、返り値は書けたサイズ。  
  
cell fSeek(cell fh, cell offset)  
  
fh のファイル位置を移動（fseek/fseeko 相当）。  
  
offset の基準（先頭/現在/末尾）を固定にしている最小実装もあるし、3引数にして whence も渡す実装もある。  
  
forth-dwc の64bit化の話題で出ていたように、OS/処理系で fseek のオフセット型差が出るので、ここを“吸収層”にするのが目的。  
  
入力バッファ（TIB）・REPL起動系  
char tib[256];  
  
Text Input Buffer 的なもの。  
  
REPLで読み込んだ1行、あるいは boot() で投入する文字列などを一時的に載せる場所。  
  
addLit("argX", argv[i]) のキー名生成に流用しているのも、まさに“一時文字列バッファ”用途。  
  
void repl()  
  
Read–Eval–Print Loop の 1サイクル（または1行分）を実行する関数。  
  
典型的には：  
  
入力を tib へ  
  
>in を0に  
  
インタプリタ（interpret）を回す  
  
ok を出す  
  
“外側の無限ループ”は main() で回し、repl() は「一回分」を担当するのが定番。  
  
void boot(const char *fn)  
  
起動処理。  
  
典型的には：  
  
端末モード初期化（raw等）  
  
fn があればそのファイルを読み込み・評価（スクリプト実行）  
  
なければデフォルトの boot.fth 相当を探す、等  
  
“Forthの初期辞書はC側で作った上で、起動スクリプトで増やす”設計の要。  
  
2) main関数の内部構造と処理の進行手順  
提示された main() を、処理の意味が分かる順に分解します。  
  
ステップ0: VM/辞書の初期化  
cfInit();  
Forth VM のコアを初期化。  
  
典型的には：  
  
スタック/リターンスタック初期化  
  
辞書ポインタ初期化（HERE/LASTなど）  
  
プリミティブ登録（Cで定義されたワード群を辞書に追加）  
  
変数（base, state, >in など）初期化  
  
ステップ1: argc を Forth から参照できるようにする  
addLit("argc", (cell)argc);  
“名前→値”のリテラル（定数/変数的エントリ）を辞書に追加。  
  
Forth側で argc というワードを実行すると、スタックに argc が積まれる、みたいな設計が多いです。  
  
ステップ2: argv[i] を arg0〜arg9 として辞書登録  
strcpy(tib, "argX");  
for (int i=0; (i<argc) && (i<10); i++) {  
    tib[3] = '0' + i;  
    addLit(tib, (cell)argv[i]);  
}  
tib を一時作業バッファにして "arg0" .. "arg9" を作り、  
  
それぞれの値に argv[i]（文字列ポインタ）を登録。  
  
こうすると Forth側で arg1 を読むだけで “起動引数1のC文字列ポインタ” を得られます。  
  
その後、Forth側に cstr>string 的な変換ワードがあれば取り回せる。  
  
ステップ3: 起動スクリプト実行（オプション）  
boot((1<argc) ? argv[1] : 0);  
引数が1つ以上（= argv[1] がある）なら、それを“起動ファイル名”として boot() に渡す。  
  
つまり ./cf hello.fth 的に起動すると、最初に hello.fth をロードして評価する想定。  
  
引数が無ければ boot(0)。  
  
この場合、boot() 側が「デフォルトファイルを探す」か「何もしない」か「起動メッセージだけ」かは実装次第。  
  
ステップ4: 無限に REPL を回す  
while (1) { repl(); }  
repl() が「1回の対話単位（1行 or 1コマンド列）」を処理し、戻ってきたらまた次へ。  
  
ここで repl() が例外/エラーを内部で握りつぶして ok に戻す設計だと、常時対話が続けられる。  
  
ステップ5: 形式上のreturn  
return 0;  
実際は無限ループなので通常到達しない。  
  
補足：この構造の“整理ポイント”が見える観点  
system.c を整理するなら、機能別にセクション分けすると一気に読みやすくなります。  
  
端末層：ttyMode/qKey/key/emit/zType  
  
時間層：ms/timer  
  
FS層：fOpen/fClose/fRead/fWrite/fSeek  
  
REPL層：tib/repl/boot  
  
main：cfInit と “引数→辞書注入” と “boot→repl”  
  
必要なら、次のステップとして 「system.c の並び替え案（見出しコメント付き）」 や、addLit("argX", argv[i]) が“文字列ポインタ”である点を踏まえて Forth側で扱いやすくする受け口（例：arg1$ のように長さ付き文字列にする） の設計案も一緒に出せます。  
  
あなた:  
  
cf.cで出てくる関数についても、役割の説明リストを教えてください。  
  
  
static void push(cell x)   
static cell pop()   
static void rpush(cell x)   
static cell rpop()   
static void comma(cell n)   
static int  changeState(int newState)   
static void checkWS(char c)   
  
static int nextWord()  
static DE_T *addWord(char *w)   
static DE_T *findWord(const char *w)   
static void compileNumber(cell n)  
void addLit(char *name, cell val)  
static void cfInner(cell pc)  
static int isNumber(const char *w)  
static void executeWord(DE_T *dp)   
static void compileWord(DE_T *dp)  
static int isStateChange()   
void cfOuter(const char *src)  
void cfInit()  
  
ChatGPT:  
了解です。提示の cf.c はだいたい「Forth の中核（スタック/辞書/インタプリタ/コンパイラ）」が入っていて、関数は大きく (A) スタック, (B) 辞書/コンパイル, (C) 外側インタプリタ, (D) 内側インタプリタ に分かれます。以下、役割リストです。  
  
A. データスタック / リターンスタック  
static void push(cell x)  
  
データスタックへ 1セル push。  
  
Forth の演算は基本ここに積んで進む。  
  
static cell pop()  
  
データスタックから 1セル pop。  
  
演算やワード実行時に引数を取り出す。  
  
static void rpush(cell x)  
  
リターンスタックへ push。  
  
DO..LOOP の制御値、復帰先、内側インタプリタの戻り管理などに使う。  
  
static cell rpop()  
  
リターンスタックから pop。  
  
B. 辞書（Dictionary）とコンパイル支援  
static void comma(cell n)  
  
「辞書領域（HERE）へ 1セル書き込み、HERE を進める」関数。  
  
Forth の ,（comma）のC実装に近い。  
  
コンパイル時にトークン/アドレス/即値などを連結していく核。  
  
static DE_T *addWord(char *w)  
  
新しいワード（辞書エントリ）を辞書に追加し、DE_T*（dictionary entry）を返す。  
  
名前を保存し、リンク（last）を更新し、コードフィールド等を初期化する役。  
  
static DE_T *findWord(const char *w)  
  
辞書を探索して、名前 w に一致するワードを探す（見つかれば DE_T*、無ければNULL）。  
  
外側インタプリタが “入力トークンがワードか？”を判断する要。  
  
static void compileNumber(cell n)  
  
数値リテラル n を「実行時にスタックへ積むコード」として辞書にコンパイルする。  
  
典型：LIT（もしくは doLit 的なプリミティブ）と n を comma() で並べる。  
  
void addLit(char *name, cell val)  
  
name という “ワード名” を作り、その実行時効果が「val を push」になるよう登録するユーティリティ。  
  
main() 側で argc や arg0.. を作るのに使ってたのがこれ。  
  
C. 入力のトークン化 / 外側インタプリタ支援  
static void checkWS(char c)  
  
whitespace 判定（スペース/タブ/改行など）。  
  
入力から単語を切り出す nextWord() の補助。  
  
static int nextWord()  
  
入力バッファ（例：tib）と >in を参照しつつ、次の “単語” を取り出す。  
  
取り出した単語をどこかのワーク領域に置き、成功/失敗（入力終端）を返す形式が多い。  
  
static int isNumber(const char *w)  
  
w が数値として解釈できるか判定する（base も参照しがち）。  
  
返り値は「数値なら真、違えば偽」＋実値は別変数/グローバルに入れる実装が多い。  
  
static int isStateChange()  
  
直前に読んだ単語が「STATE を変えるものか？」を判定する補助。  
  
例：: でコンパイルモードへ、; でインタプリタモードへ、[ ] で一時的切替…など。  
  
“即時語(immediate)” をどう扱うかにも絡む。  
  
static int changeState(int newState)  
  
state 変数を newState に切り替える共通処理。  
  
付随処理（例えば : のときヘッダ作成を開始、; のとき EXIT を付けて確定、など）をここに寄せている構造も多い。  
  
void cfOuter(const char *src)  
  
外側インタプリタ（Outer Interpreter）。  
  
src（入力文字列）を tib 等に置き、nextWord() で単語を読む→  
  
findWord() で見つかれば「実行 or コンパイル」  
  
見つからなければ isNumber() で数値解釈し「push or compileNumber」  
  
どちらでもなければエラー  
  
“Forthっぽい対話処理の本体”。  
  
D. 実行（内側インタプリタ）と「実行/コンパイルの分岐」  
static void cfInner(cell pc)  
  
内側インタプリタ（Inner Interpreter）。  
  
pc（threaded code のプログラムカウンタ相当）を受け取り、  
トークン列を順に取り出して対応するプリミティブ/ワードを動かす。  
  
token-threaded / direct-threaded / subroutine-threaded で実装は変わるが、役割は「実行ループ」。  
  
static void executeWord(DE_T *dp)  
  
辞書エントリ dp を「今すぐ実行」する。  
  
プリミティブならC関数へ、コロン定義なら cfInner() に入る、などを振り分ける。  
  
static void compileWord(DE_T *dp)  
  
辞書エントリ dp を「辞書へコンパイル」する。  
  
通常語なら “呼び出しトークン” を comma() で入れる。  
  
即時語なら compile中でも executeWord() しちゃう、などのルールがここに絡む。  
  
E. 初期化  
void cfInit()  
  
VM全体の初期化：スタック初期化、辞書領域初期化、主要変数初期化、プリミティブ登録など。  
  
system.c の main() が最初に呼ぶ「エンジンの起動」。  
  
まとめ（読み方のコツ）  
cfOuter() が「テキストを読む・解釈する（REPLやbootが呼ぶ先）」  
  
cfInner() が「コンパイル済みコードを走らせる（コロン定義の実行ループ）」  
  
その間を繋ぐのが executeWord/compileWord/compileNumber/addWord/findWord です。  
  
必要なら次に、あなたが今作っている MSX-DOS .COM の token-threaded VM（forth_cf_msx_demo.asm）と対応づけて、  
  
cfOuter 相当がどこ  
  
cfInner 相当がどこ  
  
comma/compileNumber/compileWord がどのように「トークン列」を作るか  
  
を “C版 cf.c → Z80 asm 版 VM” の対照表にして整理できます。  
  
あなた:  
  
#define PRIMS(X) \  
	X(DUP,     "dup",     t=TOS; push(t); ) \  
	X(SWAP,    "swap",    t=TOS; TOS=NOS; NOS=t; ) \  
	X(DROP,    "drop",    pop(); ) \  
  
について再度教えてください。  
  
Cで記述されたWORDのすべてが「#define PRIMS(X) \」で定義されて、用途に応じて  
　　#define X1(op, name, code) op,  
　　#define X2(op, name, code) NCASE op: code  
　　#define X3(op, name, code) { op, name, 0 },  
で利用されていると理解しています。  
Cで記述されたCFのWORDはこのPRIMSで記述されたものですべてでしょうか？cf-boot.fthやdisk.xfは外部ファイルとして読み込まれて、PRIMS(X)に追加されていくのでしょうか？  
ChatGPT:  
理解はかなり合っています。ポイントは、  
  
PRIMS(X) は「Cで実装された プリミティブ(最下層ワード) の一覧表」  
  
cf-boot.fth や disk.xf は「起動時に読み込まれて“辞書にワードを追加する”データ」  
  
外部ファイルが PRIMS(X) に“追加される”わけではない  
  
…です。  
  
1) PRIMS(X) で定義されるのは「Cプリミティブ」全部か？  
あなたの言う通り、典型的にはこの1つのリストから  
  
X1(op,name,code) op, → enum の opcode 一覧  
  
X2(op,name,code) case op: code → 内側インタプリタの switch 実行本体  
  
X3(op,name,code) { op, name, 0 }, → “opcode ↔ 名前” の表（辞書初期化用）  
  
を生成します。  
  
なので **「Cでワードとして公開したい“プリミティブ”は、基本すべて PRIMS に並べる」**設計が多いです。  
  
ただし注意点が2つあります。  
  
注意A：Cに関数があっても「ワード」じゃないものがある  
push/pop/findWord みたいな補助関数は C で書かれていても Forthのワードではありません。  
ワードとして公開されるのは「辞書に登録され、名前を持つもの」です。  
  
注意B：例外的に PRIMS 以外で登録するワードも“あり得る”  
例えば、  
  
addWord() を直接呼んで「特殊ワード」を個別登録する  
  
“状態遷移”(: ; [ ]) を PRIMS ではなく outer interpreter の特別扱いで処理する  
  
みたいな実装もあります（あなたが挙げていた isStateChange()/changeState() がまさにその匂い）。  
  
つまり結論としては：  
  
「Cで実装された“プリミティブ”は通常 PRIMS が中心」  
  
でも : などが PRIMS に入っていない方式もあり得る（outer側の特別処理）  
  
PRIMS 以外の“C関数”は、ワードではなく内部部品のことも多い  
  
2) cf-boot.fth や disk.xf は PRIMS に追加されるの？  
追加されません。  
PRIMS(X) は コンパイル時に固定の「Cプリミティブ一覧」です。  
  
cf-boot.fth などの外部ファイルは、起動時に boot() → cfOuter() などを通して  
  
: 〜 ; の コロン定義  
  
constant / variable 的な 高レベル定義  
  
include による追加ロード  
  
といった形で **辞書に新しいエントリ（DE_T）を“動的に追加”**します。  
  
つまり **“PRIMSに追加”ではなく、“辞書メモリに追加”**です。  
  
3) じゃあ「CFのWORD」は PRIMS だけで全部なの？  
いいえ。（たいていの場合）  
  
PRIMS = 最小カーネル（dup swap drop + - みたいな基礎）  
  
cf-boot.fth = その上に積む“標準っぽい語彙”や便利語（2dup, over, . の周辺、制御構造の糖衣など）  
  
disk.xf = 名前から推測すると、さらに語彙を増やすための **“外部辞書イメージ/バイナリ形式”**または 追加スクリプトのどちらか（ここはあなたの cf 実装の実物次第）  
  
なので「ユーザーが使える全ワード」は通常  
  
PRIMS（Cプリミティブ） + 起動時にロードしたForth定義（.fth 等） +（あれば）辞書イメージ（.xf等）  
  
の合成になります。  
  
4) disk.xf は何をするもの？  
手元の cf 実装を見ていないので断定は避けますが、よくあるパターンは2つです。  
  
辞書の保存イメージ（高速起動）  
いちいち boot.fth を解釈コンパイルし直さず、すでに構築済み辞書をロードする方式。  
  
追加ワード群の“スクリプト”（拡張Forthソース）  
拡張語彙をまとめてロードする。  
  
どちらでも PRIMS に追加するのではなく、辞書領域を増やす/復元する系です。  
  
もし disk.xf の先頭数十バイト（ASCIIっぽい/バイナリっぽい）や、boot() の中で disk.xf をどう扱っているかのコード（fOpen/fReadしてる部分）を貼ってくれれば、「これは辞書イメージ方式か、Forthソース方式か」をはっきり判定して説明できます。  
  
あなた:  
void cfInit()の中のstruct { char *nm; cell val; } nvp[]のCFで登録されたFORTHワード変数になるのでしょうか？これは最後が{ 0 ,0 }となっていることから、後から定義して増えていくものかと思います。  
ChatGPT:  
とても良い所に気づいています 👍  
結論から言うと、  
  
*cfInit() 内の  
struct { char *nm; cell val; } nvp[] に書かれているものは、  
「CF起動時に あらかじめ登録される FORTH の“変数・定数ワード”」です。  
  
ただし  
「後から自動的に増えていく仕組み」ではありません。  
増えるのは 辞書（DE_T）側 で、nvp[] 自体は固定テーブルです。  
  
以下、順を追って整理します。  
  
1️⃣ nvp[] は何者か？  
典型的には cfInit() の中に、こんな配列があります（簡略）：  
  
static struct {  
    char *nm;  
    cell  val;  
} nvp[] = {  
    { "base",   (cell)&base },  
    { "state",  (cell)&state },  
    { ">in",    (cell)&toIn },  
    { "here",   (cell)&here },  
    { "last",   (cell)&last },  
    { 0, 0 }  
};  
これは何か？  
👉 **「C 側の変数（ポインタ）を、Forth から触れる“名前付きワード”として登録するための初期テーブル」**です。  
  
2️⃣ CF/Forth から見ると、これは「変数ワード」？  
はい、Forth から見ると「変数ワード」になります。  
  
たとえば：  
  
base  
を実行すると、  
  
base というワードが  
  
C の変数 base のアドレスを  
  
スタックに積む  
  
という動作になります。  
  
つまり Forth 的には：  
  
base @      \ 現在の基数を読む  
10 base !   \ 基数を10進にする  
が可能になるわけです。  
  
🔑 重要ポイント  
  
nvp[] にあるのは「値」ではなく “アドレス（ポインタ）”  
  
だから @ / ! で C 側の変数と直結する  
  
3️⃣ nvp[] の {0,0} は「後から増える」ため？  
ここは 誤解しやすいポイント ですが、  
  
{0,0} は  
「後から増える」ためではなく、  
「C の for ループで走査を止めるための終端マーカー」  
  
です。  
  
典型コードはこんな感じです：  
  
for (i = 0; nvp[i].nm; i++) {  
    addLit(nvp[i].nm, nvp[i].val);  
}  
nm == 0 になったら終了  
  
配列サイズを別に持たなくて済むための C の定番イディオム  
  
➡ nvp[] 自体は静的・固定  
  
4️⃣ では「後から増える FORTH 変数」はどこへ？  
辞書（DE_T）です。  
  
例  
variable foo  
を実行すると：  
  
addWord("foo") が呼ばれ  
  
辞書に 新しい DE_T が追加され  
  
メモリ上にセル領域が確保される  
  
これは：  
  
nvp[] とは無関係  
  
実行時にどんどん増える  
  
5️⃣ nvp[] と addLit() の関係  
cfInit() ではたいていこうなっています：  
  
for (i=0; nvp[i].nm; i++) {  
    addLit(nvp[i].nm, nvp[i].val);  
}  
addLit(name, val) の意味は：  
  
「実行すると val をスタックに積むワードを作る」  
  
なので：  
  
nvp エントリ	Forthからの見え方  
{ "base", &base }	base → C変数 base のアドレス  
{ "here", &here }	here → 辞書ポインタのアドレス  
{ "argc", argc }	argc → 数値定数  
6️⃣ 全体像（重要）  
┌──────────────┐  
│ PRIMS(X)     │  ← Cプリミティブ（dup, swap, +, …）  
└──────────────┘  
        │  
┌──────────────┐  
│ nvp[]        │  ← C変数をForthに公開（base, state, >in, here…）  
└──────────────┘  
        │  
┌──────────────┐  
│ boot.fth     │  ← 高レベルForth語彙（: ; if do loop …）  
└──────────────┘  
        │  
┌──────────────┐  
│ user scripts │  ← ユーザー定義  
└──────────────┘  
7️⃣ まとめ（核心）  
✅ nvp[] に書かれたものは CF起動時に登録される FORTH の変数（または定数）  
  
❌ nvp[] 自体が後から増えることはない  
  
✅ 後から増えるのは 辞書（DE_T）  
  
🔑 {0,0} は 配列終端マーカー  
  
🔑 C のグローバル変数と Forth の @ ! が 直結する設計  
  
この構造を理解できているのは、  
Z80/MSX で CF を再構成するうえで「核心部分」を掴めている証拠です。  
  
次に進むなら、  
  
nvp[] を MSX-DOS 版 CF でどう表現するか  
  
base/state/>in を Z80 側 RAM にどう割り当てるか  
  
addLit() を token-threaded VM の LIT にどう落とすか  
  
このあたりが自然な次の一歩になります。  
  
あなた:  
では  
enum { STOP, LIT, JMP, JMPZ, NJMPZ, JMPNZ, NJMPNZ, PRIMS(X1) };  
を教えてください。  
ChatGPT:  
では、この行を CF の実行モデル（token-threaded VM）そのものとして、噛み砕いて説明します。  
  
enum { STOP, LIT, JMP, JMPZ, NJMPZ, JMPNZ, NJMPNZ, PRIMS(X1) };  
これは 「CF 仮想マシンが理解できる“命令コード（opcode）一覧”」 を定義しています。  
  
1️⃣ これは何を定義しているのか（1行要約）  
CF の inner interpreter（cfInner()）が switch(op) で処理する  
“すべての命令コード（VM命令）”の番号表  
  
です。  
  
Forth 的に言うと：  
  
スレッドコード（compiled code）に並ぶ“トークン”の正体  
  
Z80 版で言えば JP I_DUP などに対応する番号  
  
2️⃣ enum の前半：VM制御用の「特別命令」  
STOP,     // 0  
LIT,      // 1  
JMP,      // 2  
JMPZ,     // 3  
NJMPZ,    // 4  
JMPNZ,    // 5  
NJMPNZ,   // 6  
🔹 STOP  
実行終了  
  
cfInner() のループを抜けるための命令  
  
コロン定義の最後やエラー脱出に使われる  
  
Z80 VM なら：  
  
I_STOP:  
    ret  
🔹 LIT  
次のセルを即値として push  
  
Forth の「数値リテラル」の核  
  
コンパイル例：  
  
123  
→ スレッドコード：  
  
LIT 123  
🔹 JMP  
無条件ジャンプ  
  
制御構造の基礎（IF/BEGIN/AGAIN など）  
  
JMP addr  
🔹 JMPZ / JMPNZ  
条件付きジャンプ  
  
TOS を pop して判定する系  
  
命令	意味  
JMPZ	TOS == 0 ならジャンプ  
JMPNZ	TOS != 0 ならジャンプ  
🔹 NJMPZ / NJMPNZ  
“N”付きジャンプ（最適化用・省pop型が多い）  
  
実装により：  
  
pop しない  
  
NOT 条件  
  
next-token 前提  
  
などの違いがあります。  
  
👉 MSX/Z80 版を作るなら、最初は無視してもよい命令群です  
（あとで最適化として入れる位置）  
  
3️⃣ 後半：PRIMS(X1) の正体  
ここが Xマクロの核心です。  
  
X1 の定義（あなたの前提）  
#define X1(op, name, code) op,  
PRIMS(X1) 展開結果（例）  
enum {  
    STOP,  
    LIT,  
    JMP,  
    JMPZ,  
    NJMPZ,  
    JMPNZ,  
    NJMPNZ,  
  
    DUP,  
    SWAP,  
    DROP,  
    ADD,  
    SUB,  
    ...  
};  
👉 PRIMS に並んだ “すべての C プリミティブ” が opcode として enum に入る  
  
4️⃣ なぜ enum なのか（設計意図）  
① スレッドコードが「整数列」になる  
辞書に並ぶコードは：  
  
[DUP][DUP][ADD][STOP]  
→ 実体は：  
  
int code[] = { DUP, DUP, ADD, STOP };  
② inner interpreter が高速になる  
switch (op) {  
    case DUP:   ... break;  
    case ADD:   ... break;  
}  
Cコンパイラが jump table を作れる  
  
関数ポインタより軽い  
  
👉 token-threaded VM の王道  
  
5️⃣ PRIMS(X1) が「Cで書かれたワードすべて」か？  
ここで重要な整理です。  
  
✅ PRIMS(X) に入るもの  
Cで実装された “プリミティブワード”  
  
dup swap drop + - emit key など  
  
opcode を持つ  
  
❌ PRIMS に入らないもの  
: ; [ ] など 構文制御  
  
variable constant など 辞書操作語  
  
それらは：  
  
outer interpreter の特別処理  
  
あるいは boot.fth で定義  
  
6️⃣ cf-boot.fth は enum を増やすのか？  
増やしません。  
  
enum { … PRIMS(X1) };  
は Cコンパイル時に確定。  
  
cf-boot.fth は：  
  
: を使って  
  
既存 opcode（DUP, JMP, LIT など）を  
  
並べ直して新しいワードを作る  
  
だけです。  
  
👉 新しい“語彙”は増えるが、新しい“命令”は増えない  
  
7️⃣ Z80 / MSX 版に置き換えると  
この enum はそのまま：  
  
CF VM の命令セット仕様書  
になります。  
  
Z80 では：  
  
C enum	Z80 VM  
LIT	I_LIT  
DUP	I_DUP  
JMP	I_JMP  
STOP	I_EXIT  
あなたがやっている  
  
8086 / C → Z80 マクロ化  
token-threaded VM の再構築  
  
に 完全に対応する設計です。  
  
8️⃣ まとめ（核心）  
enum { STOP, LIT, JMP, …, PRIMS(X1) };  
  
= CF 仮想マシンの命令セット  
  
前半：制御命令（VM用）  
  
後半：PRIMS = Cプリミティブ命令  
  
外部 .fth は 命令を増やさず、並べ替えるだけ  
  
Z80/MSX 版では そのまま opcode 表になる  
  
次に自然に進むなら：  
  
switch(op) の PRIMS(X2) 展開  
  
PRIMS(X3) が 辞書初期化にどう使われるか  
  
LIT / JMP / JMPZ が Forth の IF/DO/LOOP にどう対応するか  
  
どれから行きますか？  
  
あなた:  
色々教えいただいてありがとうございます。  
まだ頭の中で整理中であります。今の私の理解では以下のようになります。正しいでしょうか？  
　・Cで記述されたCFのコアな部分では起動後に内容が追加されるのは定義されたスタック領域とワード変数だけである。  
　・Cで記述されたCFのコアな部分は通常は変更されず、外部の.fthだけが状況に合わせて修正されていく。  
ChatGPT:  
とても良い整理の仕方です。  
結論から言うと、**あなたの理解は「本質的に正しい」**です。  
ただし、1点だけ補足すると、より正確になります。  
  
以下、「✔ 正しい点」「⚠ 補足するとより正確な点」に分けて説明します。  
  
①  
Cで記述されたCFのコアな部分では  
起動後に内容が追加されるのは  
定義されたスタック領域とワード変数だけである。  
  
✔ 本質的に正しい  
「Cで書かれた CF のコア」は、  
  
VMの命令セット（enum / PRIMS）  
  
スタック（DS / RS）  
  
辞書領域（HERE, LAST）  
  
内側・外側インタプリタ  
  
を提供するだけで、  
  
起動後に“自動的に増えるC側の構造体”はありません。  
  
⚠ 補足（ここが1段深い理解）  
正確には、起動後に増えるのは：  
  
辞書エントリ（DE_T）  
  
.fth による : 定義  
  
variable, constant  
  
データ領域（HERE が進む）  
  
コンパイルされたトークン列  
  
変数の確保領域  
  
つまり、  
  
「スタック領域が増える」  
  
というよりは、  
  
「辞書とその背後のメモリが増える」  
  
がより正確な表現です。  
  
🔑 スタックは  
  
“伸び縮み”はするが  
  
“サイズや構造が増える”わけではない  
  
という点だけ補足しておくと完璧です。  
  
②  
Cで記述されたCFのコアな部分は通常は変更されず、  
外部の.fthだけが状況に合わせて修正されていく。  
  
✔ これは完全に正しい（しかも思想レベルで）  
これは Forth／CF設計思想の核心です。  
  
C側 = 不変の仮想マシン  
  
.fth = 可変の言語・環境・アプリ  
  
という役割分担。  
  
なぜこの設計が強いか？  
VMは 小さく・安定  
  
言語拡張は Forth自身で行う  
  
移植時（Z80 / MSX / FPGA）も  
  
C側（or ASM側）をほぼそのまま  
  
.fth を使い回せる  
  
👉 あなたが今やっている  
「C版CFをZ80/MSXに写像する」作業は  
まさにこの思想を正しくなぞっています。  
  
③ 図で整理すると（最終形）  
┌──────────────────────┐  
│ Cで書かれたCFコア     │ ← 通常は固定  
│                      │  
│  - VM命令(enum/PRIMS)│  
│  - データ/RSスタック │  
│  - 辞書管理           │  
│  - inner / outer     │  
└─────────┬────────────┘  
          │  
          ▼  
┌──────────────────────┐  
│ 起動時に増えるもの    │  
│                      │  
│  - nvp[]の変数ワード  │  
│  - boot.fth の定義   │  
│  - includeされた語彙 │  
│                      │  
│ → すべて辞書に追加   │  
└─────────┬────────────┘  
          │  
          ▼  
┌──────────────────────┐  
│ ユーザー定義 (.fth)   │ ← 状況に応じて変更  
│                      │  
│  - 新しいワード       │  
│  - アプリ             │  
│  - 実験               │  
└──────────────────────┘  
④ 一言で言うと（あなたの理解の完成形）  
CFとは  
「Cで書かれた固定VMの上に、  
Forth自身が自分の言語と環境を積み上げていく系」  
である。  
  
これはもう 実装者の視点です。  
  
次に自然につながる問い（今は答えなくてOK）  
Z80/MSX 版では  
どこまでを“Cコア相当”として固定にするか？  
  
PRIMS の最小集合はどこか？  
  
boot.fth をどこまで削れるか？  
  
token-threaded と direct-threaded の境界は？  
  
今は「整理中」とのことなので、  
この理解が腑に落ちたら、次はもう設計フェーズです。  
  
焦らず、今の整理はとても健全です。  
  
あなた:  
自分でsystem.cとcf.cを眺めていて感じたことは、これらのsystem.cとcf.cはCFというFORTHシステムを形作る仮の部品の集合体であって、CFの実体はmain()が実行されたときにはじめて姿を見せると感じました。この表現は合っていますか？  
ChatGPT:  
はい――  
その表現は とても本質を突いています。  
  
しかもそれは「ユーザー視点」ではなく、  
実装者視点の理解です。  
  
少しだけ整理して、より正確な形にしてみましょう。  
  
あなたの表現  
system.c と cf.c は  
CF という FORTH システムを形作る仮の部品の集合体であって、  
CF の実体は main() が実行されたときに初めて姿を見せる。  
  
結論  
✅ 概念的には正しい。  
ただし、  
🔧 「仮の部品」というより  
👉 **「静的な機構」**と言うとさらに正確です。  
  
なぜその感覚が生まれるのか  
1️⃣ コンパイル時  
system.c / cf.c には：  
  
命令セット（enum）  
  
PRIMS  
  
push/pop  
  
inner interpreter  
  
outer interpreter  
  
辞書管理  
  
スタック領域  
  
が書かれています。  
  
でもこの時点では：  
  
辞書はまだ空  
  
スタックはまだ空  
  
boot.fthも未実行  
  
ユーザー定義も無い  
  
👉 つまり「エンジンだけ」の状態です。  
  
2️⃣ main() が呼ばれる瞬間  
cfInit();  
addLit(...);  
boot(...);  
while(1) repl();  
ここで初めて：  
  
辞書が初期化され  
  
Cプリミティブが登録され  
  
nvp[] が登録され  
  
boot.fth が読み込まれ  
  
コロン定義が生成され  
  
HERE が進み  
  
実際の言語体系が形成される  
  
👉 この瞬間に初めて「CFというForth」が立ち上がる  
  
もっと深く言うと  
CFの実体は：  
  
静的コード（C）＋ 動的辞書メモリ  
の合成体です。  
  
Cコードだけではまだ「可能性」に過ぎません。  
  
Forthは：  
  
実行時に自分自身を組み立てる言語  
  
だからあなたは  
  
「main()が走ったときに姿を見せる」  
  
と感じたわけです。  
  
それは非常に正確な直感です。  
  
別の言い方をすると  
system.c / cf.c は：  
  
構造体  
main() 実行後は：  
  
生きたシステム  
あるいはもっとForth的に言えば：  
  
静的なVM + 動的な辞書 = Forth  
さらに一段深い視点（重要）  
実は、  
  
CFの「本体」は Cコードではない  
  
とも言えます。  
  
本体は：  
  
辞書メモリ（threaded codeの連鎖）  
です。  
  
inner interpreter は単なる実行装置。  
  
あなたが今作っている Z80 版でも、  
  
Cが消えて  
  
Z80になっても  
  
「辞書とトークン列」があれば  
それがCFの本体になります。  
  
まとめ  
あなたの理解は：  
  
✅ 概念的に正しい  
✅ 実装レベルでも正しい  
✅ Forthの哲学にも合っている  
  
より正確に言うなら：  
  
CFはCコードの集合体ではなく、  
main() 実行時に構築される「動的辞書を持つ仮想機械」である。  
  
今あなたが見えている景色は、  
  
「Forthを使っている人の視点」ではなく  
**「Forthを作っている人の視点」**です。  
  
とても良いところに来ています。  
  
もしよければ次に考えると面白い問い：  
  
では Z80/MSX版で  
「main() に相当する瞬間」はどこか？  
  
そこに答えが出ると、  
あなたのVM設計は一段完成します。  
  
あなた:  
「では Z80/MSX版で  
「main() に相当する瞬間」はどこか？」  
  
考えてみましたが、かなりむつかしい問いかけに感じます。  
実体がないものから実体が現れるとすると本当にコンパイラになってしまうのでしょうが、FORTHは違うと思います。CFが立ち上がるとメッセージと入力待ち状態になって、キー入力された文字列からは辞書の登録か実行が行われます。  
FIG＿FORTHを調べていた時に、起動時に、「初期設定」→「ABORT」→「QUIT」と進んで終わってしまうのが不思議でした。ところが、実は「QUIT」の中は無限ループになっていて、その中で「キー入力」→「INTERPRET」でキーボードから入力されたWORD名や数値などが実行されていく。もしもエラーが発生すると、エラー処理の終わった後で「QUIT」が実行されて、（まるで何事も無かったのかのように）キー入力待ちに戻ってしまう。  
すごくすっきりしているが、実際の動きがつかみにくい、不思議な感覚でいます。  
ChatGPT:  
その「不思議な感覚」、まさに Forth の起動モデルの核心です。あなたが書いた FIG-FORTH の流れ（初期設定→ABORT→QUIT、でも QUIT の中が無限ループ）は、**“実体がないものから実体が現れる”というより「最小の実行機構が常駐していて、そこに言語が積み上がる」**という発想にぴったり合っています。  
  
あなたの問いに答える形で、Z80/MSX 版での **「main() に相当する瞬間」**を、Forth らしく“見える形”に落とします。  
  
1) 「main() に相当する瞬間」＝“QUIT に入る瞬間”に近い  
C版 CF でいう main は：  
  
cfInit()（VMの基礎が動ける状態になる）  
  
boot()（起動語彙を読み込んで辞書が育つ）  
  
repl() を回す（入力待ち→解釈→実行）  
  
でしたね。  
  
FIG-FORTH ではこれが：  
  
初期設定（スタック、辞書ポインタ、I/O、変数）  
  
ABORT（状態を整えて、例外復帰点を作って）  
  
QUIT（無限ループの対話世界へ入る）  
  
になる。  
  
つまり、  
  
Z80/MSX 版で「CFが姿を見せる瞬間」＝最初に QUIT（あるいはそれ相当の外側ループ）に入る瞬間  
  
です。  
  
“コンパイラが起動する”というより、**「永遠に動き続ける対話ループが起動する」**感じ。  
  
2) 「実体がない → 実体が現れる」の正体は “辞書が育つ” こと  
Forth はコンパイラというより、もっと素朴に言うと：  
  
常駐しているのは：  
  
スタック  
  
内側インタプリタ（実行機構）  
  
外側インタプリタ（入力を単語に切る＋見つけて実行/コンパイル）  
  
辞書（最初は小さい）  
  
起動後に「育つ」のは：  
  
辞書（コロン定義、変数、定数、便利語）  
  
この「辞書が育つ」ことで、ユーザーが見ている **“Forthという世界”**が立ち上がります。  
  
だからあなたの言う  
  
実体がないものから実体が現れる  
  
は、より正確には  
  
最小の核（VM）があり、辞書が育つことで実体（言語環境）が現れる  
  
です。  
  
3) なぜ「ABORT → QUIT」で終わったように見えるのか  
これは Forth の美しさ（＝トリック）で、実際はこうです：  
  
ABORT は **「環境を初期状態に戻す」**ワード  
  
スタックを空に  
  
state をインタプリタに  
  
base や >in を整える  
  
例外復帰先を設定する（実装により）  
  
QUIT は **「対話ループに入る」**ワード  
  
入力  
  
interpret  
  
ok  
  
そしてまた入力…  
  
で、エラーが起きると：  
  
エラー処理（メッセージ表示など）  
  
ABORT と同等の“整える処理”へ飛ぶ  
  
また QUIT に戻る  
  
なので、見た目は  
  
何事もなかったように入力待ちへ復帰  
  
になります。  
  
これは「プログラムが終わらない」ための設計で、OS の main loop と同じです。  
  
4) Z80/MSX版に落とすと「main相当」はこう書ける  
あなたの MSX-DOS .COM の世界観に直すと、だいたいこういう段階になります：  
  
(A) 初期化（= cfInit 相当）  
SP/RS の初期化  
  
変数（base/state/>in/here/last 等）の初期化  
  
プリミティブ辞書の登録（PRIMS相当）  
  
画面/BDOS I/O の準備  
  
(B) ブート（= boot / cf-boot.fth 相当）  
もしブートスクリプトがあるなら読み込んで解釈  
  
MSX-DOS なら OPEN/READ して cfOuter に食わせる  
  
あるいは「最小ブート」を内蔵して : や ; 等だけ成立させる  
  
(C) QUIT ループ（= repl の無限ループ）  
入力待ち（BDOSの1文字入力 or 行入力）  
  
トークン化  
  
辞書探索  
  
実行 or コンパイル  
  
エラーなら ABORT 的に戻す  
  
また入力待ち  
  
✅ この (C) に “初めて入った瞬間” が、あなたが感じた **「CFが姿を見せた瞬間」**です。  
  
5) 「すっきりしているが動きがつかみにくい」の正体  
つかみにくさの原因はシンプルで、  
  
Cや一般言語：main → 関数呼び出しの一本道  
  
Forth：**常に QUIT の世界（無限ループ）**で、そこから  
  
入力が来るたびに世界が少しずつ変わる（辞書が育つ）  
  
エラーも “例外で落ちる” のではなく “世界の入口に戻る”  
  
つまりプログラムの“形”が、  
  
「一本道」ではなく「円環」  
  
なんです。  
  
6) すっきり理解するための超短い合言葉  
CFの本体 = 辞書  
  
CFの心臓 = inner interpreter  
  
CFの顔 = QUIT（外側ループ）  
  
エラー復帰 = ABORT → QUIT に戻る  
  
もし、あなたの forth_cf_msx_demo.asm の現在の構造（start〜実行〜終了）を、この FIG-FORTH 的な  
  
INIT  
  
ABORT  
  
QUIT（無限ループ）  
  
に“対応づけ”して図にしてみましょうか。  
Z80側で「QUITの内側（入力→INTERPRET）」を最小で作ると、急に全体が掴めるようになります。  
  
あなた:  
「いきなり全体を作り上げるにはどこから手を付ければいいのだろう？」と思っていました。対応付けした図を見せていただけますか？  
ChatGPT:  
もちろん。ここでは C版CF（system.c/cf.c） と、あなたの Z80/MSX-DOS .COM 版（forth_cf_msx_demo.asm） を、FIG-FORTH 的な  
  
INIT → ABORT → QUIT（無限ループ）  
（QUITの中で：KEY入力 → INTERPRET → 実行/コンパイル → ok → さらにKEY）  
  
に 対応付けした図として見せます。  
  
対応付けの全体図（C版CF ⇔ Z80/MSX版）  
C版CF（mainが“儀式”をやる）  
main()  
 ├─ cfInit()           … VMの核を初期化（スタック/辞書/PRIMS登録）  
 ├─ addLit(arg...)     … 変数/定数ワードを追加（argc,arg0..）  
 ├─ boot(file?)        … 起動スクリプト読み込み（boot.fth等）  
 └─ while(1) repl()    … REPL/QUITループ（入力→解釈→実行→ok）  
Z80/MSX-DOS版（startが“儀式”をやる）  
ORG 0100h  
JP start  
  
start:  
 ├─ INIT相当            … SP/RS/変数/辞書/HERE/LAST/PRIMS準備  
 ├─ BOOT相当            … （任意）起動スクリプト相当を読む/内蔵する  
 └─ QUIT相当(loop):     … 無限ループ  
      ├─ KEY/ACCEPT      … 入力（1文字 or 1行）  
      ├─ INTERPRET       … トークン化→辞書探索→実行/コンパイル  
      ├─ ok表示          … プロンプト  
      └─ エラー時ABORT   … 状態を整えてQUITへ戻る  
FIG-FORTHの「INIT→ABORT→QUIT」を“機能”で分解した対応図  
ここが肝です。FIG-FORTHを「何をしているか」で割ると、Z80でも作りやすくなります。  
  
【INIT】（一度だけ）  
  - データスタック初期化  
  - リターンスタック初期化  
  - 変数初期化（BASE, STATE, >IN, HERE, LAST…）  
  - 辞書の最初の形を作る（PRIMS登録/ヘッダ構造準備）  
  - I/O初期化（BDOS/表示/キー入力）  
  
        ↓  
  
【ABORT】（いつでも呼ばれる“復帰儀式”）  
  - データスタックを空に  
  - STATEをインタプリタへ  
  - >INを0へ  
  - 必要ならエラーメッセージを表示  
  - （例外復帰点があればそこへ）  
  
        ↓  
  
【QUIT】（無限ループ）  
  loop:  
    - 入力を得る（ACCEPT/KEY）  
    - INTERPRET（単語→見つける→実行 or コンパイル）  
    - ok を出す  
    - goto loop  
**Z80/MSX版の “main相当の瞬間” は、この QUIT loop に「最初に入った瞬間」**です。  
  
もう一段具体化：Z80/MSX .COM の「箱割り図」  
あなたの forth_cf_msx_demo.asm を、次の “箱” に分けると、どこから作るべきかが見えます。  
  
┌────────────────────────────────┐  
│ [A] start / INIT                │  ← cfInit相当  
│   - SP/RS設定                   │  
│   - 変数領域初期化              │  
│   - 辞書初期化(HERE/LAST)       │  
│   - PRIMSテーブル登録           │  
│   - 画面に起動メッセージ         │  
└───────────────┬────────────────┘  
                │  
┌───────────────▼────────────────┐  
│ [B] ABORT（復帰ルーチン）        │  
│   - DSを空に / STATE=0          │  
│   - >IN=0 / エラーなら表示       │  
│   - QUITへジャンプ              │  
└───────────────┬────────────────┘  
                │  
┌───────────────▼────────────────┐  
│ [C] QUIT（外側ループ）           │ ← repl相当  
│   loop:                         │  
│     - ACCEPT/KEYで入力取得       │  
│     - INTERPRET                 │ ← cfOuter相当  
│     - ok表示                    │  
│     - JP loop                   │  
└───────────────┬────────────────┘  
                │  
┌───────────────▼────────────────┐  
│ [D] INTERPRET（外側インタプリタ）│ ← cfOuter  
│   - nextWord（単語切り出し）      │  
│   - findWord（辞書探索）          │  
│   - stateで分岐：                │  
│      * 実行: executeWord         │  
│      * compile: compileWord/LIT  │  
│   - 数値なら LIT or push         │  
└───────────────┬────────────────┘  
                │  
┌───────────────▼────────────────┐  
│ [E] INNER（内側インタプリタ）    │ ← cfInner  
│   - token列を実行（switch相当）  │  
│   - PRIMS（dup/swap/drop等）     │  
│   - STOPで復帰                  │  
└────────────────────────────────┘  
「どこから手を付けるべきか？」の実装順（迷いに効く順番）  
全体を一気に作らず、**“動く最小環”**を先に作ると不思議が消えます。  
  
ステップ1：QUITの骨格だけ作る（入力→エコー）  
BDOSで1文字読み、1文字出す（echo）  
  
これで 「待って戻る」ループが手に入ります  
（= QUITの雰囲気が出る）  
  
ステップ2：nextWord相当（単語に切る）  
スペース区切りで単語を拾う（tibと>in）  
  
まだ辞書無しでOK  
  
ステップ3：最小辞書（数語だけ）＋ findWord  
dup drop . みたいな数語だけ  
  
見つかったら「識別できた」と表示するだけでもOK  
  
ステップ4：innerを繋ぐ（token実行）  
ここで初めて PRIMS と cfInner 相当が意味を持つ  
  
“辞書エントリ→実行コード”が流れ出す  
  
ステップ5：ABORTを入れる  
エラーでも 必ずQUITに戻る  
  
あの「何事もなかったように戻る」体験が完成します  
  
ひとことで言うと  
あなたが感じている「不思議さ」は、  
  
“一本道のプログラム”ではなく、  
“入口（QUIT）に常に戻る世界”  
  
だからです。  
  
そして Z80/MSX版では、  
  
QUIT loop を先に作ると、全体が急に掴める  
  
ようになります。  
  
もしよければ、あなたの forth_cf_msx_demo.asm の現在のラベル構成（start 以降の主要ラベル名だけでOK）を貼ってください。  
この図の [A]〜[E] を **あなたの実ファイルのラベルに直接対応付けた“あなた専用の図”**にして返します。  
  
あなた:  
その前に質問ですが、私がFIG-FORTHを調べて理解した「ABORT」→「QUIT」の流れですが、CFではその部分はどのような構造になっているのでしょうか？  
ChatGPT:  
いい質問です。あなたの FIG-FORTH の理解（ABORT → QUIT で“何事もなかったように入力待ちへ戻る”）は、CFでも ほぼ同じ思想で実現されています。ただし CF は FIG-FORTH のように「ABORT というワード名」が前面に出るというより、C 側の構造（repl/cfOuter の戻り方）として ABORT/QUIT 相当を作っていることが多いです。  
  
あなたが貼ってくれた CF の main() から見える範囲だけで、対応関係をまとめます（※手元にあなたの実 repl() / cfOuter() の中身が無いので、ここは CF 系で典型の構造として説明します。違いがあれば、該当関数を貼ってくれれば「あなたのCFの実装そのもの」に即して描き直せます）。  
  
CFでの「QUIT 相当」はどこ？  
CFでは while(1){ repl(); } が、ほぼそのまま **QUIT（外側ループ）**です。  
  
FIG-FORTH: QUIT = 無限ループで「入力→INTERPRET→また入力」  
  
CF: main() の無限ループ + repl() がそれを担当  
  
イメージ：  
  
main:  
  cfInit()  
  boot(...)  
  while(1) repl()    ← これが QUIT世界（外側ループ）  
repl() の中でだいたいこういうことをやります：  
  
入力を tib に入れる（ACCEPT 相当）  
  
cfOuter(tib) を呼ぶ（INTERPRET 相当）  
  
ok を出す  
  
戻る（main の while がまた repl を呼ぶ）  
  
CFでの「ABORT 相当」はどこ？  
CF の ABORT 相当は、普通は「1つの関数名」ではなく、2層に分かれて現れます。  
  
レベル1：軽いABORT（状態を整えて続行）  
repl() が 毎回の入力の前後で、  
  
スタックを整える（空にする、あるいは underflow を回復）  
  
state=0（インタプリタモード）  
  
>in=0  
  
途中コンパイルを破棄（必要なら）  
  
みたいな“復帰処理”を入れて、次の入力待ちに戻す。  
  
これは FIG-FORTH の ABORT がやる仕事と同じです。  
  
レベル2：強いABORT（例外脱出）  
CF実装によっては、エラー時に  
  
setjmp/longjmp  
  
あるいは cfOuter() がエラーコードを返して repl() が復帰処理へ  
  
のどちらかで、深い入れ子（実行中）から一気に REPL へ戻る構造を持つことがあります。  
  
FIG-FORTH の “ABORT が QUIT に戻る” の「戻り方」に近いのは、この強いABORTです。  
  
CFを FIG-FORTH の ABORT→QUIT に対応させると  
あなたのCFは、見えている関数名だけで対応付けるとこうです：  
  
FIG-FORTH  
INIT → ABORT → QUIT(無限)  
              └─ ACCEPT → INTERPRET → …  
CF  
cfInit → boot → mainのwhile(1) { repl(); }  
                        └─ (ACCEPT) → cfOuter(tib) → …  
                              └─ エラーなら（ABORT相当の復帰処理）  
つまり CF では、  
  
QUIT = main の while + repl  
  
INTERPRET = cfOuter  
  
ABORT = repl（または cfOuter）のエラー復帰処理  
  
になっている、という理解がいちばんスッキリします。  
  
「何事もなかったように戻る」感じは CF でも出る？  
出ます。理由も同じです。  
  
エラーは「プロセス終了」ではなく「対話ループへ戻る」ために設計されている  
  
結果として、ユーザーは 次の入力待ち（ok） をまた見る  
  
ただし、CFの実装によっては FIG-FORTH ほど“ABORTという儀式感”が表に出ず、  
  
cfOuter() がエラー表示して return  
  
repl() が ok を出して次へ  
  
のように、Cの制御構造として自然に見える形になっていることが多いです。  
  
ここから先を「あなたのCFの実装そのもの」で確定させるには  
次のどちらか（できれば両方）の中身を、短くでいいので貼ってください：  
  
void repl() の中身  
  
void cfOuter(const char *src) の中身（特にエラー時の処理）  
  
そうすれば、  
  
CFの「ABORT相当」が どの変数をリセットしているか  
  
エラー脱出が return型か longjmp型か  
  
“compile中のエラー” がどう復帰するか  
  
を、FIG-FORTHの ABORT/QUIT と 1対1対応の図で確定版として描けます。  
  
  
