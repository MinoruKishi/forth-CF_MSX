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
void cfInit()の中のstruct { char *nm; cell val; } nvp[]のCFで登録されたFORTHワード変数になるのでしょうか？これは最後が{ 0 ,0 }となっていることから、後から定義して増えていくものかと思います。
では
enum { STOP, LIT, JMP, JMPZ, NJMPZ, JMPNZ, NJMPNZ, PRIMS(X1) };
を教えてください。
色々教えいただいてありがとうございます。
まだ頭の中で整理中であります。今の私の理解では以下のようになります。正しいでしょうか？
　・Cで記述されたCFのコアな部分では起動後に内容が追加されるのは定義されたスタック領域とワード変数だけである。
　・Cで記述されたCFのコアな部分は通常は変更されず、外部の.fthだけが状況に合わせて修正されていく。
自分でsystem.cとcf.cを眺めていて感じたことは、これらのsystem.cとcf.cはCFというFORTHシステムを形作る仮の部品の集合体であって、CFの実体はmain()が実行されたときにはじめて姿を見せると感じました。この表現は合っていますか？
「では Z80/MSX版で
「main() に相当する瞬間」はどこか？」

考えてみましたが、かなりむつかしい問いかけに感じます。
実体がないものから実体が現れるとすると本当にコンパイラになってしまうのでしょうが、FORTHは違うと思います。CFが立ち上がるとメッセージと入力待ち状態になって、キー入力された文字列からは辞書の登録か実行が行われます。
FIG＿FORTHを調べていた時に、起動時に、「初期設定」→「ABORT」→「QUIT」と進んで終わってしまうのが不思議でした。ところが、実は「QUIT」の中は無限ループになっていて、その中で「キー入力」→「INTERPRET」でキーボードから入力されたWORD名や数値などが実行されていく。もしもエラーが発生すると、エラー処理の終わった後で「QUIT」が実行されて、（まるで何事も無かったのかのように）キー入力待ちに戻ってしまう。
すごくすっきりしているが、実際の動きがつかみにくい、不思議な感覚でいます。
「いきなり全体を作り上げるにはどこから手を付ければいいのだろう？」と思っていました。対応付けした図を見せていただけますか？
その前に質問ですが、私がFIG-FORTHを調べて理解した「ABORT」→「QUIT」の流れですが、CFではその部分はどのような構造になっているのでしょうか？
