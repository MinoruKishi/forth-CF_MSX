# GitHub MinoruKishi/forth-CF_MSX   
  
## CCurl/CFを初めて読んだ人がChatGPTの力を借りてのMSX0向けFORTH改造の顛末記〜０６  
  
# あなた:  
  
cf-main.zip  
  
Zip アーカイブ    
CFというFORTHシステムを理解しようとしています。cf.cの中に次の行があります。  
  
`static DE_T *addWord(char *w) {`  
  
ここのDE_Tとは何でしょうか？  
  
# ChatGPT:  
  
良いところに注目されています 👍    
結論から言うと、  
  
DE_T は CF における「辞書エントリ（Dictionary Entry）」を表す構造体型    
です。  
  
---  
  
## **DE_T の正体（cf.h より）**  
  
cf.c ではなく cf.h に定義があります。  
  
~~~
typedef struct {  
  
   cell xt;          *// 実行トークン (execution token)*  
  
   byte flags;       *// フラグ（IMMEDIATE / INLINE など）*  
  
   byte len;         *// ワード名の長さ*  
  
   char name[NAME_MAX+1]; *// ワード名*  
  
} DE_T;  
~~~
  
つまり DE_T は：  
  
~~~
Forth の 1 ワードを C 構造体として表したもの  
~~~
  
です。  
  
---  
  
## **各フィールドの意味（Forth視点）**  
  
| フィールド | 意味 | Forth 的解釈 |  
| ----- | ----- | ----- |  
| xt | 実行アドレス | そのワードの本体（CODE/コロン定義の入口） |  
| flags | 属性 | IMMEDIATE / INLINE など |  
| len | 名前長 | 比較高速化用 |  
| name | ワード名 | 辞書に登録される文字列 |  
  
これはまさに Forth の辞書エントリそのもの です。  
  
---  
  
## **addWord() で DE_T が何をしているか**  
  
質問に出てきた関数です：  
  
`static DE_T *addWord(char *w)`  
  
この関数は：  
  
1. 新しい Forth ワードを定義するときに    
2. 辞書領域に DE_T を 1 個確保し    
3. そのポインタを返す  
  
という役割を持っています。  
  
### **実際の処理（重要部分）**  
  
~~~
last -= sizeof(DE_T);          *// 辞書を後ろに伸ばす*  
  
DE_T *dp = (DE_T*)last;        *// そこを DE_T とみなす*  
  
dp->xt = (cell)here;           *// 実行コードの開始位置*  
  
dp->flags = 0;  
  
dp->len = ln;  
  
strcpy(dp->name, w);  
  
return dp;  
~~~
  
👉 典型的な「メモリ直置き辞書」実装です    
（colorForth / Tachyon 系らしい設計）  
  
---  
  
## **CF の辞書構造の特徴（あなた向け補足）**  
  
あなたはすでに  
  
* forth-DWC    
* FIG-Forth    
* 8086 / Z80 / 6502 Forth  
  
を深く触っておられるので、重要な違いだけ整理します。  
  
### **❌ ないもの**  
  
* LINK フィールド（単方向リスト）    
* NAME フィールドの可変長構造    
* ヘッダ直後にコードが続く配置  
  
### **✅ あるもの**  
  
* 固定長 DE_T の配列    
* 線形探索（findWord()）    
* xt は「C 上のアドレス」ではなく VM の code[] 上の番地  
  
つまり CF の DE_T は、  
  
「古典的 Forth 辞書を C 構造体 + フラット配列 に落とし込んだもの」  
  
と考えると非常に理解しやすいです。  
  
---  
  
## **findWord() との関係**  
  
`static DE_T *findWord(const char *w)`  
  
は、  
  
~~~
DE_T *dp = (DE_T*)cw;  
  
if ((ln == dp->len) && strEqI(dp->name, w)) { return dp; }  
~~~
  
という形で DE_T を順番に舐めて探すだけです。  
  
👉 DE_T が「辞書の最小単位」であることがここでも分かります。  
  
---  
  
## **まとめ（超重要）**  
  
* DE_T = Dictionary Entry Type    
* CF における 1ワード = 1 DE_T    
* Forth の「名前＋XT＋属性」を C 構造体にしたもの    
* addWord() は「辞書に DE_T を追加する関数」  
  
---  
  
次に進むならおすすめは：  
  
* DE_T と forth-DWC の NVP_T / DE_T 比較    
* FIG-Forth の (name, link, code) との対応関係    
* なぜ CF が LINK を捨てたか（＝ colorForth 的思想）  
  
このあたり、とても面白い話になります 😊    
続けますか？  
  
# あなた:  
  
質問です。私はCFをZ80のアセンブラコードで書き換えられないかを調べています。CFの辞書の構造をFIGforthで教えてもらった、ひとつのワードの塊の中にデータとアセンブラの混在した辞書変換部と実行部が一体になったforth辞書の構造で書き換えられないでしょうか？  
  
# ChatGPT:  
  
できます。結論から言うと CF の「DE_T 配列辞書」を、FIG-Forth 風の「ワードごとにヘッダ＋実行部が一体」の辞書に作り替えるのは可能です。    
ただし “そのまま移植” ではなく、CF が捨てている要素（LINK など）を復活させる設計変更になります。  
  
---  
  
## **まず整理：CF と FIG-Forth の辞書は「分離」vs「一体」**  
  
### **CF（あなたが見ている設計）**  
  
* ワード情報：固定長 DE_T を別領域に配列的に積む    
  * name / len / flags / xt がまとまった“メタ情報テーブル”    
* 実行本体：xt が指す先（たぶん code[]/here 側）に置く    
* 探索：DE_T を線形に舐める（リンク無しでも成立）  
  
### **FIG-Forth（古典的）**  
  
* 1ワードが 辞書内に一塊で置かれる    
* 概念的にはこんな並び（実装流儀により微差あり）：  
  
[LINK] [NAME(長さ+文字列)] [FLAGS(即時など)] [CFA/XT] [PFA...]  
  
* 探索：LINK を辿って名前比較（新しい単語から逆順）  
  
---  
  
## **「CF を FIG 風に」する時の肝：DE_T を辞書ヘッダに埋め込む**  
  
CF で DE_T が担っている要素は、FIG だと辞書ヘッダに入ります：  
  
| CF の DE_T | FIG 辞書ヘッダに対応 |  
| ----- | ----- |  
| name,len | NAME フィールド |  
| flags | IMMEDIATE/SMUDGE 等のフラグ |  
| xt | CFA/XT（コード入口） |  
| （無い） | LINK（前の単語へのポインタ） |  
  
つまり、DE_T を捨てて、その内容を“辞書ヘッダ”として各ワードの前に埋め込むのが基本方針です。  
  
---  
  
## **Z80 での具体イメージ（定番のレイアウト例）**  
  
Z80 なら、ポインタは 16bit で済むので、辞書はこういう感じにできます（例）：  
  
~~~
; ---- Dictionary entry ----  
  
dw  link        ; 前のワードへのポインタ (0 なら終端)  
  
db  flags_len   ; 上位ビットにフラグ、下位に長さ(0..31など)  
  
db  'D','U','P' ; 名前文字列（長さ分）  
  
; ここでアライン/パディングしても良い  
  
dw  cfa         ; コードフィールドアドレス（実行入口）  
  
; PFA は cfa の先、または cfa が DOCOL などを指して PFA に続く
  
;  （注）PFAやCFAについては、つたない解説ではありますが、以下を参照願います。
;　　　https://github.com/MIN0/ToUnderstandForth/blob/main/JPN/Documents/03-02_v0.21.01_JPN/03_02_03_Process_flow_of_ToUnderstandFORTH.MD  
  
~~~
  
### **そして「実行部」は FIG 流儀で分岐**  
  
* プリミティブ（機械語）：cfa がそのまま Z80 ルーチン入口    
* コロン定義：cfa が DOCOL を指し、PFA にスレッド（XT列）が並ぶ    
  * NEXT が IP を進めながら XT を実行していく、あの形です  
  
---  
  
## **“混在した辞書変換部と実行部が一体”は、Z80 だとむしろやりやすい**  
  
あなたが言う「データ（名前・リンク・PFA）とアセンブラ（実行部）が同じ塊」というのは、まさに FIG の良さで、  
  
* Z80 アセンブラなら dw db でヘッダを書きつつ    
* 直後にラベル付きルーチン（プリミティブ）を置ける    
* または DOCOL/スレッド列を置ける  
  
ので、CF より素直です。  
  
---  
  
## **ただし注意点：CF の “実装の楽さ” を捨てることになる**  
  
CF は DE_T を別にしてるから、こういう利点があります：  
  
* 名前表が固定長で扱いやすい（C で楽）    
* 辞書領域とコード領域を分けて管理できる    
* 文字列の扱いが簡単（strcmp）  
  
FIG 方式にすると代わりに：  
  
* 可変長 NAME の扱いが必要    
* LINK の辿り・比較の処理が必要    
* （実装によっては）アライン調整やヘッダ計算が面倒  
  
…とはいえ、Z80 のアセンブラで Forth を書くなら FIG 方式は王道なので、ここは「面倒」というより「正攻法」ですね。  
  
---  
  
## **移植の現実的な進め方（おすすめ）**  
  
### **ステップ1：CF の** DE_T **を「FIG ヘッダ出力」に置き換える**  
  
CF の addWord 的な処理を、Z80 版では  
  
* HERE（辞書ポインタ）にヘッダを書き込む    
* LATEST（最新ワードの先頭）を更新する    
* FIND は LATEST から LINK を辿る  
  
に変えます。ここで DE_T は消滅。  
  
### **ステップ2：実行系は “間接スレッド” でまず動かす**  
  
Z80 で一番移植しやすいのは  
  
* NEXT + IP + W（作業レジスタ）で回す    
* XT は 16bit で dw の列    
* プリミティブは jp (hl) 的に飛ぶ（実装流儀あり）  
  
まずここを固めると、その後の最適化（直接スレッド、インライン等）がやりやすいです。  
  
### **ステップ3：CF の “flags” を FIG フラグに再マップ**  
  
CF の flags（IMMEDIATE/INLINE など）があるなら、    
FIG でも flags_len の上位ビットに割り当てれば同じ概念が使えます。  
  
---  
  
## **どっちが良いか（あなたの目的に合わせた結論）**  
  
* **「CF を Z80 に“移植”したい」**なら：CF 方式（DE_T 配列）を残すほうが近い    
  → ただし Z80 では C みたいに strcmp も無いので、結局 FIND 周りは手間    
* **「Z80 アセンブラで Forth を“作りたい”」**なら：FIG 方式（ヘッダ一体型）が自然    
  → あなたの言う「混在した一体辞書」はこっち  
  
あなたの文章からは後者（Z80で書き換えたい・調べたい）が強いので、FIG 方式へ設計変更して再構成が一番筋が良いです。  
  
---  
  
もしよければ、次のどちらかで一気に具体化できます（質問しません、両方出せます）：  
  
1. Z80 用 FIG 辞書ヘッダの“最小実装”（LINK/NAME/XT、FIND、CREATE、:, ; まで）    
2. CF の DE_T を基準に、**「CF→FIG 変換の対応表」**と、どの関数をどう置き換えるか（addWord/findWord など）  
  
どちらの形で進めたいですか？（もちろん「両方」でもOKです）  
  
##### **あなた:**  
  
そこで気になるのが、レジスタの構成をどうしようかということです。Z80のレジスタだけで実現するにはレジスタ数が少なすぎます。以下がヒントになるのかと思いましたがどうでしょうか？  
  
~~~
#define TOS           dstk[dsp]    
#define NOS           dstk[dsp-1]    
#define L0            lstk[lsp]    
#define L1            lstk[lsp-1]    
#define L2            lstk[lsp-2]    
#define isTemp(w)     ((w[0]=='t') && btwi(w[1],'0','9') && (w[2]==0))    
#define X1(op, name, code) op,    
#define X2(op, name, code) NCASE op: code    
#define X3(op, name, code) { op, name, 0 },  
~~~
  
###### **ChatGPT:**  
  
そのヒント、役に立つ部分と、レジスタ問題とは別の部分が混ざっています。整理するとかなり見通しが良くなります。  
  
---  
  
## **そのマクロ群が示している本質**  
  
### **1)** TOS / NOS / L0..L2 **は「レジスタ」じゃなくて「メモリ上のスタック」を前提にしている**  
  
~~~
#define TOS dstk[dsp]  
  
#define NOS dstk[dsp-1]  
  
#define L0  lstk[lsp]  
~~~
  
...  
  
これは **「Z80のレジスタが足りないなら、そもそもスタックは配列（メモリ）で持つ」**という CF の設計思想をそのまま表しています。  
  
* dstk[] = データスタック（配列）    
* dsp = データスタックポインタ（配列の添字）    
* lstk[] = ループ/ローカル用スタック（配列）    
* lsp = そのポインタ  
  
つまり、レジスタ不足を、最初からメモリスタックで回避している設計です。    
Z80でも同じで、レジスタに全部載せる必要はありません。  
  
---  
  
## **Z80で「レジスタが少ない」問題の現実的な解き方**  
  
Z80でForth VMを作る定番はこれです：  
  
### **方針A：スタック本体は Z80 のハードSP を使う（高速）**  
  
* データスタック = Z80 SP    
* リターンスタック = IX/IY など別ポインタ＋メモリ（または逆にする）  
  
ただし CF は「配列＋添字」設計なので、CF互換を意識するなら次の方針が自然です。  
  
### **方針B：CFの通り「スタックは配列」で持ち、レジスタは“キャッシュ”に使う（堅い）**  
  
* dsp / lsp はメモリ上の変数（16bit）    
* dstk / lstk は連続メモリ領域    
* Z80レジスタは「今触ってる値」を一時的に持つだけ  
  
これなら レジスタ数が少なくても成立します。  
  
---  
  
## **具体的なレジスタ割り当て（おすすめ・実戦形）**  
  
CFのVM（オペコード実行ループ）をZ80でやるなら、まずこれが安定です：  
  
* HL = IP（インストラクションポインタ。code列の位置）    
* DE = W / XT（次に実行するワード/ハンドラのアドレス）    
* BC = 一時（演算・アドレス計算）    
* IX = &dstk[dsp]（トップ要素のアドレスを指す）    
* IY = &lstk[lsp]（ループ/ローカルのトップ）  
  
ポイントは「値そのもの」より “ポインタ” をレジスタに置くことです。    
Z80は 16bit レジスタでアドレス計算が強いので、これが効きます。  
  
### **例：TOS/NOS を “アドレス” で扱う**  
  
* IX が dstk[dsp] を指していれば    
  * TOS = (IX+0..1)    
  * NOS = (IX-2..-1) みたいにアクセス可能  
  
（※ dstk のセル幅が16bitなら2バイト刻み）  
  
---  
  
## **“TOS/NOS をレジスタに載せたい”場合の最小キャッシュ**  
  
さらに速くしたいなら定番の「2要素キャッシュ」：  
  
* HL = TOS 値（16bit）    
* DE = NOS 値（16bit）    
* IX は dsp の位置（または &dstk[dsp]）  
  
こうすると + や AND のような2項演算が速いです。  
  
ただしこの方式は、PUSH/POPのたびにメモリと同期する規約が必要になります。    
（でもForth VMではよくやります）  
  
---  
  
## **その残りのマクロは「レジスタ問題」とは別物**  
  
### **2)** isTemp(w) **は “テンポラリ変数名の判定” のため**  
  
#define isTemp(w) ((w[0]=='t') && btwi(w[1],'0','9') && (w[2]==0))  
  
これは例えば t0〜t9 みたいな一時変数（テンポラリ）を特別扱いする用途で、    
レジスタ割り当てそのものを示すものではありません。  
  
ただし発想としてはヒントになります：  
  
“よく出る一時値は t0..tN に落として、レジスタっぽく扱う”  
  
Z80でも「擬似レジスタ」を メモリ上の固定スロット（t0..t9） として持つのは強いです。  
  
### **3)** X1/X2/X3 **は opcode テーブル自動生成のため**  
  
~~~
#define X1(op,name,code) op,  
  
#define X2(op,name,code) NCASE op: code  
  
#define X3(op,name,code) { op, name, 0 },  
~~~
  
これは典型的な “Xマクロ” で、  
  
* 列挙型（opcode番号）    
* switch-case の本体    
* 逆引きテーブル（名前など）  
  
を 同じ定義から一括生成する仕組みです。  
  
これは Z80移植でも超有用で、同じことができます：  
  
* 命令表（オペコード→実装ラベル）のテーブル生成    
* ディスアセンブル用の名前表生成    
* ジャンプテーブル生成  
  
でも、レジスタ数を増やす効果は直接はないです。  
  
---  
  
## **結論：あなたの懸念への答え**  
  
* Z80のレジスタだけで全部やろうとすると足りません（正しい感覚）    
* でも CF 自体が スタックを配列（メモリ）で持つ設計なので、Z80でも同様にやれば成立します    
* そのマクロ群のうち、TOS/NOS/L0.. は「メモリスタック前提」の強いヒント    
* isTemp と X1..X3 はレジスタではなく、**実装の整理（テンポラリ扱い / テーブル生成）**のヒントです  
  
---  
  
もし次に進めるなら、あなたの目的（CFに近いVMか、FIG風のスレッディドか）に合わせて、  
  
* (1) CF方式：opcode VM を Z80で回すレジスタ割り当て（NEXT相当含む）    
* (2) FIG方式：IP/W/NEXT のZ80最小実装（データスタックはSPか配列か選択）  
  
のどちらか（または両方）を、Z80アセンブリの雛形付きで出せます。どちら寄りで行きますか？  
  
# あなた:  
  
まだ漠然としていて、ざっくりとした内容でいいので、（１）と（２）を見せてもらえますか？  
  
# ChatGPT:  
  
了解です。ざっくり版で、**(1) CF方式（opcode VM）**と **(2) FIG方式（スレッディド）**を、Z80での「最低限のレジスタ構成」と「NEXT/ディスパッチ」の雛形までまとめて見せます。    
（細部は後であなたのCF/FIG選好に合わせて詰められます）  
  
---  
  
## **(1) CF方式：opcode VM（switch-case 相当）を Z80で回す**  
  
### **ねらい**  
  
* CFの PRIMS / Xマクロ 的な「オペコード集合」を、    
* Z80で ジャンプテーブル（または比較分岐）で実装    
* スタックは **メモリ配列（dstk/lstk）**で持つ（CFの思想そのまま）  
  
### **レジスタ割り当て（ざっくり）**  
  
* HL = IP（次のopcodeを読む位置）    
* A = opcode    
* DE = 一時 / アドレス計算    
* BC = 一時    
* IX = &dstk[dsp]（TOSの“アドレス”キャッシュ）    
* IY = &lstk[lsp]（L0の“アドレス”キャッシュ、必要なら）  
  
重要：Z80レジスタに「値」を詰め込むより、スタック頂点のアドレスを指させる方が楽で堅いです。  
  
### **ディスパッチ（ジャンプテーブル）雛形**  
  
~~~
; HL = IP  
  
NEXT:  
  
   ld a,(hl)          ; A = opcode  
  
   inc hl             ; IP++  
  
   ; jump table: opcode 0..255 を想定  
  
   ld e,a  
  
   ld d,0  
  
   add de,de          ; *2 (16bit address table)  
  
   ld bc,OPJT  
  
   add hl,bc          ; ★注意: HLはIPなので退避が必要  
  
↑ これは「HL=IPのまま足す」と壊れるので、実戦ではこうします（定番）：  
  
; HL=IP  
  
NEXT:  
  
   ld a,(hl)  
  
   inc hl  
  
   push hl            ; IP退避  
  
   ld e,a  
  
   ld d,0  
  
   add de,de          ; word table  
  
   ld hl,OPJT  
  
   add hl,de  
  
   ld e,(hl)  
  
   inc hl  
  
   ld d,(hl)  
  
   ex de,hl           ; HL = handler addr  
  
   pop de             ; DE = IP  
  
   ex de,hl           ; HL=IP, DE=handler → だと面倒  
  
   ; もっと簡単にするなら：  
  
   ; pop hl で IP復帰して、jp (ix) のように飛ぶ形にする  
~~~
  
ざっくり見せる目的なので、一番分かりやすい形に寄せると：  
  
~~~
NEXT:  
  
   ld a,(hl)  
  
   inc hl  
  
   ld e,a  
  
   ld d,0  
  
   add de,de  
  
   ld ix,OPJT  
  
   add ix,de          ; IX = &OPJT[opcode]  
  
   ld e,(ix+0)  
  
   ld d,(ix+1)  
  
   ex de,hl           ; HL = handler  
  
   jp (hl)            ; handlerへ  
  
; handlerは最後に jp NEXT  
~~~
  
（IXを毎回OPJTにするのが嫌なら、OPJTベースは固定レジスタに置くなど調整します）  
  
### **スタックアクセス（配列方式）雛形**  
  
セル幅を16bitとします（Z80向けに自然）。  
  
* dstk_top = IX が指す場所（TOSのアドレス）    
* TOS = (IX+0),(IX+1)    
* NOS = (IX-2),(IX-1)  
  
例：DROP（dsp-- 相当）  
  
~~~
OP_DROP:  
  
   ; IX = &dstk[dsp]  
  
   dec ix  
  
   dec ix             ; IX -= 2  (16bit cell)  
  
   jp NEXT  
~~~
  
例：DUP（push TOS）  
  
~~~
OP_DUP:  
  
   ; IX points to TOS cell  
  
   ld a,(ix+0)  
  
   ld b,(ix+1)  
  
   inc ix  
  
   inc ix             ; make room: dsp++  
  
   ld (ix+0),a  
  
   ld (ix+1),b  
  
   jp NEXT  
~~~
  
ここまでの雰囲気が CF の TOS/NOS マクロの発想に近いです。  
  
---  
  
## **(2) FIG方式：スレッディド Forth（DOCOL/NEXT）を Z80で回す**  
  
### **ねらい**  
  
* 典型的な FIG の「辞書一体型」を作り    
* 実行は **XT列（16bit）**をIPで辿る（間接スレッドが最初は簡単）    
* プリミティブはZ80のサブルーチン、コロン定義はDOCOLで入る  
  
### **レジスタ割り当て（最小の定番）**  
  
* HL = IP（XT列を指す）    
* IX = RP（リターンスタックのポインタ）    
  * あるいは IY をRPに使う    
* SP = データスタック（※ここが“王道”で速い）    
  * ただしあなたがCFっぽくしたいなら SP をRPにして DSを配列でもOK  
  
ここでは “FIGらしさ” を出すために、  
  
* SP = DS（データスタック）    
* IX = RP    
  で見せます。  
  
### **NEXT（XTを取って飛ぶ）雛形（間接スレッド）**  
  
XTは「実行ルーチンのアドレス」だと仮定します。  
  
~~~
; HL = IP (points to next XT in the thread)  
  
NEXT:  
  
   ld e,(hl)      ; DE = XT  
  
   inc hl  
  
   ld d,(hl)  
  
   inc hl  
  
   ex de,hl       ; HL = XT, DE = IP  
  
   jp (hl)        ; jump to code for that XT  
  
   ; 各ワードの末尾で: ex de,hl ; jp NEXT などで戻る流儀もある  
~~~
  
このままだとIPの保持が面倒なので、よくあるのは IPをメモリ変数にするか、DEを常にIP専用にする規約です。ざっくり版としては：  
  
* DE = IP 固定    
* 各ワードは最後に jp NEXT  
  
にします：  
  
~~~
; DE = IP  
  
NEXT:  
  
   ld a,(de)  
  
   inc de  
  
   ld l,a  
  
   ld a,(de)  
  
   inc de  
  
   ld h,a        ; HL = XT  
  
   jp (hl)  
~~~
  
これが一番 “図式が明快” です。  
  
### **DOCOL（コロン定義に入る）**  
  
コロン定義は  
  
* 現在のIPをRPに積む    
* 新しいIPを「その定義のPFA（XT列の先頭）」にする  
  
~~~
; DE = IP  
  
; HL = PFA (thread start) をワードごとに持たせる流儀  
  
DOCOL:  
  
   ; push current IP onto return stack (IX)  
  
   dec ix  
  
   ld (ix+0),d  
  
   dec ix  
  
   ld (ix+0),e  
  
   ; set IP = PFA (仮に HL に入っているとする)  
  
   ld d,h  
  
   ld e,l  
  
   jp NEXT  
~~~
  
実際は「ワードのCFAがDOCOLで、DOCOLは自分の後ろ（PFA）をIPにする」方式が多いので、HLへPFAを渡す部分は辞書レイアウト次第です。  
  
### **例：プリミティブ（+）**  
  
DSをZ80のSPで持つ場合、TOS/NOSはスタックトップから取ります（ざっくり）：  
  
~~~
; ( n1 n2 -- n3 )  
  
PRIM_PLUS:  
  
   pop hl        ; n2  
  
   pop de        ; n1  
  
   add hl,de  
  
   push hl  
  
   jp NEXT    
~~~
---  
  
## **(1) と (2) の違いが “レジスタ数” に与える影響**  
  
* (1) CF方式（opcode VM）    
  * ループは「opcode読み → handlerへ」    
  * スタックは配列でやるとレジスタ消費が少ない    
  * その代わり handler が多く、分岐が多い    
* (2) FIG方式（threaded）    
  * ループは「XT読み → そのXTへ」    
  * DSをSPに載せるとすごく速い＆レジスタ少ない    
  * DOCOL/RP/IPの規約が要  
  
どちらもZ80で成立しますが、あなたの「辞書一体型でやりたい」方向なら、最終的には (2) が自然に気持ちいいことが多いです。    
一方、CFのソースを“見たまま追える”のは (1) です。  
  
---  
  
次の一歩として、あなたの状況に合わせて「ざっくり」をもう少し現実化できます。例えば：  
  
* (1) CF方式：OP_LIT, OP_BRANCH, OP_0BRANCH, OP_EXIT だけ実装して最小VMを回す    
* (2) FIG方式：辞書ヘッダ（LINK/NAME/FLAGS）＋FIND＋DOCOL＋EXIT だけで : test 1 2 + . ; まで動かす  
  
どっちも“最小セット”なら短いコードで形が見えます。続けてどちらを先に具体化しますか？  
  
# あなた:  
  
なるほど。辞書の構造はどうなりますか？  
  
# ChatGPT:  
  
大事なところです。ここでは (1) CF方式（opcode VM） と **(2) FIG方式（辞書一体型）**の両方で、辞書（ワード情報）の「メモリ配置」をざっくり図で示します。    
（Z80想定で 16bitセル、アドレスも16bitで説明します）  
  
---  
  
## **(1) CF方式（DE_T相当の“別置き辞書”）の辞書構造**  
  
CFの設計はざっくりこうです：  
  
* 辞書エントリ（名前/flags/xt）：固定長の配列（テーブル）    
* 実行本体（コード/スレッド/即値など）：別の領域（here側）  
  
### **メモリ配置イメージ**  
  
低アドレス                                            高アドレス  
  
+----------------------+   +--------------------------+  +------------------+  
  
| code area (HERE↑)    |   | free                     |  | DE table (LAST↓) |  
  
|  opcodes/threads     |   |                          |  |  DE0 DE1 DE2 ... |  
  
|  literals, etc       |   |                          |  |  fixed-size      |  
  
+----------------------+   +--------------------------+  +------------------+  
  
### **DE（辞書エントリ）1個の中身（Z80版に落とすなら）**  
  
CFの DE_T { xt, flags, len, name[] } を、そのままZ80でも持てます。  
  
~~~
DE:  
  
 dw xt        ; 実行先(またはthread/handlerの先頭)  
  
 db flags  
  
 db len  
  
 db name[固定長]  ; 例: 16バイト固定、または 32バイト固定  
~~~
  
#### **特徴**  
  
* FINDは「DE配列を線形に舐めて比較」    
* LINKが要らない（だから構造が単純）    
* ただし 固定長name が無駄になりやすい  
  
あなたが「CFをZ80に移植したい」なら、この構造が一番“原文に忠実”です。  
  
---  
  
## **(2) FIG方式（辞書一体型：ヘッダ＋実行部が同じ塊）の辞書構造**  
  
ここがあなたが狙っている「混在・一体」の形です。  
  
### **1ワード = 1ブロック（例）**  
  
~~~
[ LINK ] [ FLAGS+LEN ] [ NAME... ] [ (PAD) ] [ CFA/XT ] [ PFA... ]  
~~~
  
Z80でよく使う実用的なレイアウト（例）を示します。  
  
### **例A：ヘッダ直後にCFA、その後PFA（典型）**  
  
~~~
entry:  
  
 dw  link            ; 直前のワード（LATEST）へのポインタ  
  
 db  flags_len       ; 上位bits=flags, 下位bits=len  
  
 db  'D','U','P'     ; name (len bytes)  
  
 (pad to even)       ; 16bit境界に合わせたい場合  
  
 dw  cfa             ; 実行入口（プリミティブなら機械語、コロンならDOCOL）  
  
 ... pfa ...         ; コロン定義ならここにXT列、変数なら初期値など  
~~~
  
### **例B：CFAを固定（DOCOL/PRIM識別を工夫）する流儀**  
  
もう少し “Forthらしい” のは：  
  
* プリミティブ：CFA が Z80コード入口    
* コロン定義：CFA が DOCOL のアドレス（共通）    
  * PFA（CFAの次）に XT 列が並ぶ  
  
~~~
colon word:  
  
 dw link  
  
 db flags_len  
  
 db name...  
  
 (pad)  
  
 dw DOCOL  
  
 dw xt1  
  
 dw xt2  
  
 dw xt3  
  
 dw EXIT  
  
primitive word:  
  
 dw link  
  
 db flags_len  
  
 db name...  
  
 (pad)  
  
 dw prim_code_entry   ; 直接Z80ルーチンへ  
~~~
  
#### **特徴**  
  
* FINDは LATEST から LINK を辿る（新しい順）    
* >CFA >PFA の計算が必要（名前長が可変なので）    
* “辞書＝そのまま実行形” なのでZ80アセンブラで美しい  
  
あなたが「FIG風に一体化したい」なら、これが王道です。  
  
---  
  
## **FIG方式での「>CFA」をどう計算するか（ざっくり）**  
  
可変長NAMEのせいで、CFAの位置は一定ではありません。だから  
  
1. flags_len から name長を取り出す    
2. entry + header_size + name_len まで進める    
3. 16bit境界に揃える（pad）    
4. そこが CFA  
  
という処理になります。  
  
Z80的には **NAMEを“偶数長にしてパディング無し”**にすると楽です。  
  
### **例：nameを常に偶数に丸めるルール**  
  
* len が奇数なら最後に 0 を1バイト足す（比較では無視）    
* これで CFA は常に 16bit境界に来ます  
  
---  
  
## **どちらを採るべきか（あなたの方向性に沿って）**  
  
* CFを“忠実に”Z80化：CF方式（別置きDEテーブル）    
  * 実装は素直、でもForthっぽい辞書一体感は薄い    
* Z80で“Forthらしく”作る：FIG方式（辞書一体型）    
  * あなたの目的（混在・一体化）に合う    
  * FIND/ヘッダ計算の部品が必要  
  
---  
  
## **最小サンプル（FIG方式）の見た目（雰囲気）**  
  
~~~
LATEST -> DUP  
  
DUP:  
  
 dw  DROP  
  
 db  3            ; len=3 (flags=0)  
  
 db  "DUP"  
  
 db  0            ; pad (lenが奇数だから)  
  
 dw  prim_DUP     ; Z80ルーチン入口  
  
DROP:  
  
 dw  0  
  
 db  4  
  
 db  "DROP"  
  
 dw  prim_DROP  
~~~
  
こういう「辞書とZ80コードが同じソースで地続き」になります。  
  
---  
  
次を出すなら、あなたが一番欲しいのはたぶんここです：  
  
* **FIG方式の FIND（リンク辿り＋名前比較＋IMMEDIATE判定）**の最小Z80雛形    
* もしくは CF方式の DE配列を舐める FIND の最小雛形  
  
どっちも短いので、両方の「FIND」雛形も続けて出しましょうか？  
  
# あなた:  
  
なんとなくですが、私はFIGforthを最初に知ったので、あの辞書構造には強烈なインパクトがありました。ですが、今回のタイトルはforth-CF_MSXです。CF方式で行きましょう。  
  
ただ、気になっているのは、FIGforthのような辞書の中に辞書変換部と実行部が一体になったforth辞書の構造というのはあるのでしょうか？  
  
# ChatGPT:  
  
とても本質的な問いです。結論から先に言います。  
  
---  
  
## **結論（核心）**  
  
あります。    
しかもそれは珍しい例外ではなく、**Forthの歴史の中で何度も現れている“正当な系統”**です。  
  
ただし重要なのは：  
  
「CF方式（別置き辞書）」と    
「FIG方式（辞書一体型）」は    
対立概念ではない  
  
という点です。  
  
“辞書の持ち方” と “実行モデル” は直交する    
──これが答えの軸になります。  
  
---  
  
## **まず整理：あなたが言う「辞書変換部と実行部が一体」とは何か**  
  
FIG-forth の衝撃は、たぶんこれです：  
  
~~~
[ LINK ][ NAME ][ CFA ][ PFA ... ]  
~~~
  
               ↑  
  
         実行はここに飛ぶ  
  
* 名前を見つけた瞬間に    
* そのまま実行できる形になっている    
* 辞書＝実行形式、という一致感  
  
これは    
👉 「変換（compile / interpret）と実行が同じ構造を共有している」    
という感覚だったはずです。  
  
---  
  
## **CF方式は「それを壊した」のか？**  
  
いいえ。壊していません。  
  
CFは単にこう分離しただけです：  
  
~~~
[ 辞書メタ情報 ]  ← 変換用（name, flags, xt）  
  
[ 実行コード ]    ← 実行用（opcode, thread, etc）  
~~~
  
しかしここで重要なのは：  
  
**CFの xt（execution token）はすでに「実行可能な形」を指している**  
  
という事実です。  
  
FIGの CFA と、CFの xt は概念的に同じものです。  
  
---  
  
## **「辞書一体型 × CF方式」は存在するか？**  
  
### **答え：存在します**  
  
ただし名前は FIG-forth ではなく、次の系統で現れます。  
  
---  
  
## **実在する「一体型 × 非FIG」Forthたち**  
  
### **1. Subroutine Threaded Forth（STC）**  
  
* 各ワードは CPUの CALL/RET で直接呼ばれる    
* 辞書内に  
  
~~~
[ LINK ][ NAME ][ CODE ... ]  
~~~
  
という形で    
ヘッダ直後に機械語コードが置かれる  
  
👉 辞書＝そのまま実行コード  
  
* 実行モデルは CF とは違うが    
* 「一体感」は FIG 以上に強烈  
  
---  
  
### **2. Direct Threaded Code（DTC）**  
  
* 辞書内の PFA に    
  * ジャンプ命令    
  * 実行アドレス    
* が直接埋め込まれる  
  
~~~
[ LINK ][ NAME ][ JMP next ][ data... ]  
~~~
  
👉 NEXT が不要、または極小  
  
---  
  
### **3. Token Threaded × 一体辞書**  
  
これが一番あなたの問いに近いです。  
  
* 辞書は FIG 風に一体    
* しかし PFA に並ぶのは    
  * 機械語アドレスではなく    
  * opcode / token  
  
~~~
[ LINK ][ NAME ][ CFA=DOCOL ][ token ][ token ][ token ]  
~~~
  
* token → opcode → handler    
* つまり CFの opcode VM を、FIG辞書の中に埋め込む  
  
👉 「CF方式だが辞書はFIG風」  
  
これは完全に成立します。  
  
---  
  
## **ではなぜ CF は別置き辞書にしたのか？**  
  
これは C で書くため です。  
  
* 可変長ヘッダは C では面倒    
* strcmp / 配列 / 構造体の方が楽    
* メモリ管理が明快  
  
つまり CF は  
  
**Forthを壊したのではなく**    
**Cに“翻訳”した**  
  
だけです。  
  
---  
  
## **forth-CF_MSX で「一体感」を取り戻す方法（CF方式のまま）**  
  
あなたが「CF方式で行く」と決めた上でFIG的な一体感を取り戻す現実的な道はこれです。  
  
### **方法A：DE_T を “実行ブロックの先頭に置く”**  
  
~~~
[ DE_T ][ opcode / thread / literal ... ]  
~~~
  
* xt = &DE_T + sizeof(DE_T)    
* FIND したら、そのまま xt 実行  
  
👉 見た目は FIG、中身は CF  
  
---  
  
### **方法B：opcode/thread を辞書と同じ領域に積む**  
  
* DE_T は今のまま    
* code[] をやめて    
* 辞書領域に  
  
~~~
[ word1 header ][ word1 code ]  
  
[ word2 header ][ word2 code ]  
~~~
  
👉 変換と実行が同じアドレス空間  
  
---  
  
## **重要な認識（ここが一番大事）**  
  
FIG-forthの衝撃は    
「構造」ではなく    
「一致感」だった  
  
* 見つけたものが    
* そのまま    
* 実行できる  
  
CF方式でも    
xt を“即実行可能なもの”にしておけば    
この感覚は完全に再現できます。  
  
---  
  
## **まとめ（あなたの問いへの端的な答え）**  
  
* ✅ FIG-forthのように「辞書変換部と実行部が一体」の構造は存在する    
* ✅ それは FIG に限らない（STC / DTC / token-threaded でもある）    
* ✅ CF方式でも設計次第で“一体感”は取り戻せる    
* ❌ CF方式＝一体構造が不可能、ではない  
  
---  
  
もし次に進むなら、とても良い分岐点です。  
  
* CF方式のまま    
  * xt をどう設計すれば    
  * 「見つけたら即実行」の FIG感覚になるか    
* forth-CF_MSX の最小辞書レイアウト案    
  * Z80向け    
  * DE_T + opcode/thread 混在  
  
ここを一緒に設計すると、    
このプロジェクトは一段“思想的に美しく”なります。  
  
続けましょうか。  
  
# あなた:  
  
的外れなことを言ってなければいいんですが、私は次のやり方を取りたいです。  
  
「3. Token Threaded × 一体辞書」で行う。    
実行部分はZ80のコードで記述する。その時にCALL命令ではなくて、JUMP命令で書けないだろうか？    
ここは今の時点では必要ないですが、将来的にZ80以外のニーモニックで実行部を記述したい（例えばRISC-Vのアセンブラコードとか）。  
  
# ChatGPT:  
  
全然的外れじゃないです。むしろ **「Token Threaded × 一体辞書」＋「CALLなしでJUMPだけ」**は、Z80でも将来の別ISAでも通る、かなり筋の良い設計です。  
  
ポイントはこうです：  
  
* 辞書は一体型（FIGっぽい見た目）    
* 実行は token列（中間表現）    
* 実際の実行は token→(実装アドレス) のディスパッチ    
* 各プリミティブは最後に 必ず NEXT に JUMP で戻る    
* コロン定義は DOCOL 相当が IPを退避して token列へ JUMP  
  
これで CALL/RET を使わずに Forthが回ります。  
  
---  
  
## **1) Token Threaded × 一体辞書 の形**  
  
辞書エントリを「ヘッダ + 実体（token列 or 何か）」として同じ領域に置きます。  
  
例（雰囲気）：  
  
~~~
[LINK][FLAGS+LEN][NAME...][PAD][BODY...]  
~~~
  
BODY はワード種別で変えます。  
  
### **プリミティブ（Z80機械語で実装）**  
  
* BODY に「そのプリミティブを表す token（1バイト）」だけ置く    
* 実体は別にある “プリミティブ実装表” にある  
  
... [BODY:  token(PRIM_DUP)]  
  
### **コロン定義（token列）**  
  
* BODY に token を並べる    
* 最後は token(EXIT) など  
  
... [BODY: token(LIT) imm_lo imm_hi token(DUP) token(PLUS) token(EXIT)]  
  
つまり 辞書の中身は “CPU非依存の token列”。    
CPU依存なのは「token→実装」テーブルと、各プリミティブの実装だけ。  
  
---  
  
## **2) CALLなしでJUMPだけで回す（Z80）**  
  
ここが肝です。Z80だと 間接ジャンプは JP (HL) が使えるので非常にやりやすいです。  
  
### **レジスタの役割（最小の定番）**  
  
* DE = IP（現在実行中の token列の位置）    
* IX = RP（リターンスタックポインタ：メモリ上）    
* SP = DS（データスタック：Z80のSPを使うと速い）    
* HL = 作業 / ディスパッチ先  
  
（DSを配列にするCF寄り設計もできますが、ここでは「JUMPだけ」を強くするため SP=DS の王道で書きます）  
  
---  
  
## **3) 実行ループ NEXT（token→実装へJUMP）**  
  
token は 1バイトとして、JT[token] が 16bit の実装アドレスを持つ形。  
  
擬似コード（雰囲気）：  
  
~~~
; DE = IP  
  
NEXT:  
  
   ld a,(de)        ; token  
  
   inc de  
  
   ld l,a  
  
   ld h,0  
  
   add hl,hl        ; *2  
  
   ld bc,JT  
  
   add hl,bc        ; HL = &JT[token]  
  
   ld a,(hl)  
  
   inc hl  
  
   ld h,(hl)  
  
   ld l,a           ; HL = impl addr  
  
   jp (hl)          ; CALLじゃなくてJP  
~~~
  
### **プリミティブは「最後に必ず jp NEXT」**  
  
たとえば DUP：  
  
~~~
PRIM_DUP:  
  
   ; DS=SP だと TOS は (SP) 近辺から取れる（実装流儀次第）  
  
   ; ここでは雰囲気だけ  
  
   ; ... dup処理 ...  
  
   jp NEXT    
~~~
---  
  
## **4) “コロン定義”をどう動かすか（CALLなし）**  
  
コロン定義に入るときに必要なのは：  
  
1. 現在のIP(DE)をRP(IX)に積む    
2. 新しいIPを、そのワードのBODY（token列先頭）にする    
3. jp NEXT  
  
これを DOCOL 的プリミティブとして実装します。  
  
### **DOCOL（概念）**  
  
* コロン定義ワードの “実行” は token(DOCOL) でもいいし、辞書エントリに「種別=COLON」を持っておいて最初からDOCOLに飛ばしてもいい    
* いちばん単純なのは コロン定義の実行入口を DOCOL に統一して、DOCOLが “このワードのBODY先頭” を見つける方式  
  
Z80で楽なのは、ディスパッチがそのまま **DE（IP）**しか知らないので、  
  
* 「今実行しているワードのBODY先頭」を Wレジスタ（例えばHLやBC）に入れてから DOCOL に飛ぶ  
  
という規約を作ることです。  
  
#### **規約例**  
  
* PRIM_DO_COLON は HL に新IP が入っている前提  
  
~~~
PRIM_DO_COLON:  
  
   ; push current IP (DE) onto return stack (IX)  
  
   dec ix  
  
   ld (ix+0),d  
  
   dec ix  
  
   ld (ix+0),e  
  
   ; DE = new IP (HL)  
  
   ld d,h  
  
   ld e,l  
  
   jp NEXT  
~~~
  
### **EXIT（; の終わり）**  
  
* RPからIPを戻して jp NEXT  
  
~~~
PRIM_EXIT:  
  
   ld e,(ix+0)  
  
   inc ix  
  
   ld d,(ix+0)  
  
   inc ix  
  
   jp NEXT  
~~~
  
こうすると CALL/RET ゼロで、コロン定義も戻れます。  
  
---  
  
## **5) 「将来 RISC-V のニーモニックで実行部を書きたい」への答え**  
  
この設計は、そこに直結します。  
  
### **何がCPU依存で、何がCPU非依存か**  
  
* 非依存：辞書（ヘッダ）と BODY の token列、コンパイラ（tokenを吐く部分）    
* 依存：    
  * NEXT（ディスパッチの書き方）    
  * token→実装アドレス表（JT）    
  * 各プリミティブの実装  
  
RISC-V なら “間接JUMP” は jalr x0, 0(rs1)（戻り先を保存しない）で表現できます。    
つまり Z80 の jp (hl) に相当するものがちゃんとあります。  
  
### **さらに一歩：ニーモニックを差し替える方法**  
  
あなたの過去の流れ（sjasmplus＋LUAマクロ）にも相性が良いです。  
  
* 共通：PRIM_DUP, PRIM_PLUS など “意味” の名前    
* バックエンド：Z80版 / RISC-V版で、そのラベルの中身（命令列）を差し替える    
* NEXTやスタック規約もバックエンドごとに提供  
  
これで token列（中間表現）は一切変えずにCPUだけ変えられます。  
  
---  
  
## **6) ここでの設計選択（おすすめ）**  
  
あなたの方針に沿って、まずはこうすると一気に前に進みます：  
  
1. token幅：1バイト    
2. DS：Z80の SP    
3. RP：メモリ上（IXで指す）    
4. NEXT：jp (hl) の間接ジャンプ    
5. 辞書：一体型（ヘッダ＋BODY token列）    
6. コンパイラ：最小で LIT, EXIT, DOCOL, BRANCH, 0BRANCH と算術いくつか  
  
---  
  
もし次に具体化するなら、ここが最初の“設計の確定点”です：  
  
* token列の中に「即値」をどう埋めるか（LIT の後に16bitリテラルを置く、など）    
* 辞書ヘッダに「BODY先頭」をどう見つけるか（ヘッダ可変長問題の処理）  
  
この2点を決めると、Z80版の雛形（NEXT＋DOCOL＋EXIT＋LIT＋FIND＋最小辞書）をそのまま書けます。どちらも、あなたの「CF方式（実装しやすい）＋FIGの一体感（見通しが良い）」の両方を満たす形にできます。  
  
# あなた:  
  
考えを整理するために確認です。以下の二つの問題をもう少しわかりやすく説明してもらえますか？  
  
　１）token列の中に「即値」をどう埋めるか（LIT の後に16bitリテラルを置く、など）  
  
　２）辞書ヘッダに「BODY先頭」をどう見つけるか（ヘッダ可変長問題の処理）  
  
# ChatGPT:  
  
とても良い確認ポイントです。ここが整理できると、forth-CF_MSX の設計が一気に安定します。    
できるだけ「図と手順」で噛み砕いて説明します。  
  
---  
  
# **１）token列の中に「即値」をどう埋めるか**  
  
（LIT の後に 16bit リテラルを置く、とは何か）  
  
## **発想の原点（なぜ問題になる？）**  
  
Token Threaded では、  
  
* token = 実行すべき動作の番号    
* 実行系は    
  「token を読む → 対応する実装へジャンプ」  
  
という単純な世界です。  
  
ところが 数値リテラルは問題です。  
  
~~~
: TEST  123  DUP ;  
~~~
  
123 は  
  
* 「命令」ではなく    
* 「データ」    
  です。  
  
👉 token列の中に“命令ではないデータ”を混ぜる必要がある  
  
これが問題の正体です。  
  
---  
  
## **解決策の王道：LIT + 即値方式**  
  
### **基本アイデア**  
  
* 「数値そのもの」を token にしない    
* LIT（literal）という tokenを使う    
* LIT の「直後」に実際の数値データを置く  
  
---  
  
### **token列の実例（16bitセル想定）**  
  
~~~
[token_LIT] [low byte] [high byte] [token_DUP] [token_EXIT]  
~~~
  
視覚化すると：  
  
IP →  
  
+---------+---------+---------+---------+---------+  
  
|  LIT    |   0x7B  |   0x00  |   DUP   |  EXIT   |  
  
+---------+---------+---------+---------+---------+  
  
          ↑ 123 (0x007B)    
---  
  
### **実行時に何が起きるか**  
  
#### **NEXT が LIT を読む**  
  
~~~
token = LIT  
~~~
  
#### **LIT の実装がやること**  
  
1. IP が指している場所から 16bit 値を読む    
2. それを データスタックに push    
3. IP を 2バイト進める    
4. jp NEXT  
  
---  
  
### **Z80風 擬似コード（概念）**  
  
~~~
PRIM_LIT:  
  
   ; DE = IP  
  
   ld a,(de)  
  
   inc de  
  
   ld l,a  
  
   ld a,(de)  
  
   inc de  
  
   ld h,a          ; HL = literal  
  
   push hl         ; DSに積む  
  
   jp NEXT    
~~~
---  
  
### **なぜこの方式が優れているか**  
  
* token列が CPU非依存    
* 即値サイズを変えられる（16bit → 32bit も可能）    
* FIG / ITC / TTC 系で実績あり    
* RISC-V でも全く同じ構造が使える  
  
👉 「命令とデータが直列に並ぶ」    
これが Forth VM の基本構造です。  
  
---  
  
## **代替案（参考）**  
  
| 方法 | 問題点 |  
| ----- | ----- |  
| token = 数値 | token 空間が枯渇 / 可変長が扱いづらい |  
| 専用即値表 | token列と乖離する |  
| CALLで即値渡し | 今回の設計方針と矛盾 |  
  
👉 LIT + 即値が最も自然  
  
---  
  
# **２）辞書ヘッダに「BODY先頭」をどう見つけるか**  
  
（ヘッダ可変長問題とは何か）  
  
## **問題の正体**  
  
FIG型・一体辞書では 1ワードはこうなります：  
  
~~~
[ LINK ][ FLAGS+LEN ][ NAME... ][ BODY... ]  
~~~
  
ここで問題なのは：  
  
* NAME は 可変長    
* だから BODY（token列 or 実体）の開始位置が一定でない  
  
👉 FIND でワードを見つけたあと    
👉 「このワードを実行したい」    
👉 BODY の先頭アドレスを計算する必要がある  
  
これが ヘッダ可変長問題です。  
  
---  
  
## **解決策A（最もシンプル）：**  
  
### **「BODY先頭アドレスをヘッダに直接持つ」**  
  
#### **構造**  
  
~~~
[ LINK ][ FLAGS+LEN ][ NAME... ][ PAD ][ BODY_PTR ][ BODY... ]  
~~~
  
* BODY_PTR = BODY の先頭アドレス    
* FIND は    
  * 名前一致を確認    
  * BODY_PTR を読むだけ  
  
#### **長所**  
  
* 実行が簡単    
* Z80でも楽    
* CF方式との親和性が高い  
  
#### **短所**  
  
* ヘッダが少し大きくなる  
  
👉 forth-CF_MSX に一番向いている  
  
---  
  
## **解決策B（FIG原典）：**  
  
### **「FLAGS+LEN から計算する」**  
  
#### **構造**  
  
~~~
[ LINK ][ FLAGS+LEN ][ NAME... ][ PAD ][ BODY... ]  
~~~
  
#### **BODY先頭計算**  
  
1. エントリ先頭アドレスを保持    
   * 2（LINK）    
   * 1（FLAGS+LEN）    
   * len（NAME）    
   * padding（偶数境界調整）  
  
#### **長所**  
  
* ヘッダ最小    
* 正統派 FIG  
  
#### **短所**  
  
* 実装が少し面倒    
* Z80では計算が増える  
  
---  
  
## **解決策C（CF折衷）：**  
  
### **「NAME を固定長にする」**  
  
~~~
[ LINK ][ FLAGS ][ NAME(16byte固定) ][ BODY... ]  
~~~
  
* BODY は常に一定オフセット  
  
#### **長所**  
  
* 計算不要    
* 実装簡単  
  
#### **短所**  
  
* メモリ効率が悪い    
* FIG感が薄れる  
  
---  
  
## **あなたの設計に最も合う選択**  
  
あなたの条件を整理すると：  
  
* CF方式ベース    
* Token Threaded    
* 将来 RISC-V も視野    
* 実装の分かりやすさ重視  
  
👉 最適解はこれです  
  
### **✅ 解決策A：BODY_PTR を持つ一体辞書**  
  
~~~
[ LINK ][ FLAGS+LEN ][ NAME... ][ PAD ][ BODY_PTR ][ BODY(token列) ]  
~~~
  
* FIND → BODY_PTR を HL に入れる    
* DOCOL / 実行系は HL を新IPとして使う    
* CPUが変わっても辞書構造は変えなくてよい  
  
---  
  
## **２つの問題の関係（とても重要）**  
  
| 問題 | 解決の共通点 |  
| ----- | ----- |  
| 即値の扱い | token列の中に「命令とデータを直列に置く」 |  
| BODY先頭 | 辞書から「token列の先頭」を即座に得る |  
  
👉 どちらも「IP がただのポインタである」設計が鍵  
  
---  
  
## **まとめ（超重要）**  
  
### **１）即値**  
  
* LIT token の直後に 16bit 値を置く    
* LIT 実装が IP を進めてスタックに積む  
  
### **２）BODY先頭**  
  
* 一体辞書でも    
  * 計算する方法    
  * 直接ポインタを持つ方法    
* forth-CF_MSX では BODY_PTR 方式が最適  
  
---  
  
次に進むなら、ここが自然です：  
  
* forth-CF_MSX 用の「最小辞書エントリ構造」定義    
* Z80用：NEXT / LIT / DOCOL / EXIT の最小セット  
  
この2つが揃うと、    
「この設計で本当に回る」という確信が持てます。  
  
# あなた:  
  
解決案Aですが、CFでもWORDの名前の部分は    
~~~
　[ FLAGS+LEN ][ NAME... ]    
~~~
で管理しているのでしょうか？ここを一般に使われている文字列の構造を使って、以下のように変えてしまうとＦＯＲＴＨではなくなってしまうでしょうか？（用意するＷＯＲＤが変わってしまう？）    
~~~
　[ FLAGS+LEN ][ ビット７で管理しないNAME＋０ ]  
~~~
  
# ChatGPT:  
  
結論：ぜんぜん Forth じゃなくならないです。    
[FLAGS+LEN][NAME…] をやめて、**一般的な C 文字列（NUL終端）**にしても、Forthとしては問題ありません。変わるのは主に「実装の都合」と「検索・比較の仕方」で、用意するワード集合が勝手に変わるわけではありません。  
  
---  
  
## **まず、CFは名前をどう持っているのか**  
  
あなたが最初に見せてくれた DE_T は（CF系の典型として）  
  
* len（長さ）    
* name[]（固定長バッファ、末尾は \0 で終端）    
* flags  
  
を持っています。  
  
つまり CF は多くの場合、  
  
* “Forth伝統の「長さつき名前」”の雰囲気を残しつつ    
* 実装上は C 文字列（NUL終端）も同時に成立する形  
  
になっていることが多いです。  
  
なので CF 的には、あなたの案  
  
~~~
[FLAGS+LEN][ビット7で管理しない NAME + 0]  
~~~
  
はむしろ自然です。  
  
---  
  
## **「ビット7で管理」って何だったか**  
  
FIG/古典Forthには、名前の最後の文字に bit7 を立てて終端を示す流儀がありました（$80 付加みたいなやつ）。  
  
これは当時の事情（メモリ節約・比較高速化）で生まれた表現方法の一つであって、  
  
* それを使わないとForthでない    
  という性質のものではありません。  
  
---  
  
## **あなたの案にすると、何が変わる？**  
  
### **変わらないもの（Forth性）**  
  
* 辞書に「名前→実行（xt）」の対応があること    
* インタプリタが入力トークンを辞書で引いて実行/コンパイルすること    
* ワードセット（DUP, DROP, :, ;, …）は設計で決めるもの  
  
### **変わるもの（実装）**  
  
* 名前比較の方法    
  * 旧：lenとbit7終端の処理    
  * 新：'\0' まで比較（またはlenも併用）    
* ヘッダからBODY先頭を計算する方法    
  * NUL終端だと「終端まで走査→必要ならアライン」の計算になる    
  * でもあなたは解決案A（BODY_PTRを持つ）なので、ここはほぼ問題にならない  
  
---  
  
## **ただし注意：FLAGS+LEN を残す意味が薄れる**  
  
あなたの案は  
  
* NAME は ... 0 で終わる（長さは0まで数えれば分かる）  
  
なので、LENを別に持つ必然性は減ります。  
  
それでもLENを持つメリットはあります：  
  
* 比較を速くできる（まずlen一致で弾ける）    
* NAMEに 0 を含めない前提で確実に扱える    
* 大文字小文字無視比較でも扱いやすい  
  
なので実務的には、次のどっちかがスッキリします。  
  
### **案1：LENは残す（高速・簡単）**  
  
~~~
[ FLAGS ][ LEN ][ NAME...\0 ][ PAD ][ BODY_PTR ][ BODY... ]  
~~~
  
### **案2：LENを捨てる（構造が単純）**  
  
~~~
[ FLAGS ][ NAME...\0 ][ PAD ][ BODY_PTR ][ BODY... ]  
~~~
  
（この場合、FINDは毎回strlen相当が入るので少し遅い）  
  
---  
  
## **「用意するWORDが変わってしまう？」について**  
  
変わりません。    
ワードが変わるのは、**辞書に何を登録するか（ワード定義）**の問題であって、名前の表現形式（bit7終端／len付き／NUL終端）とは独立です。  
  
たとえば、  
  
* いままで DUP という名前が辞書にあった    
* 表現を NUL終端に変えた  
  
→ それでも DUP は DUP です。検索の仕方が変わるだけです。  
  
---  
  
## **おすすめ（forth-CF_MSX / 解決案Aと相性良い形）**  
  
あなたの方向性（Token threaded・将来他ISAも）なら、  
  
~~~
[ LINK(16) ][ FLAGS(8) ][ LEN(8) ][ NAME bytes ][ 0 ][ PAD ][ BODY_PTR(16) ][ BODY... ]  
~~~
  
がバランス良いです。  
  
* 互換性：Forthっぽい（lenもある）    
* 実装：C文字列にもできる（末尾0）    
* FIND：len→文字比較で速い    
* BODY先頭：BODY_PTRで即解決（可変長問題を回避）  
  
---  
  
必要なら次に、これを前提にして  
  
* 辞書1エントリの正確なバイト配置（オフセット表）    
* FINDが「len+大文字小文字無視」で比較するZ80雛形    
* 見つけたら BODY_PTR を取り出して実行へ渡す流れ  
  
を、ほんとに最小の形で書けます。  
  
# あなた:  
  
私には「おすすめ（forth-CF_MSX / 解決案Aと相性良い形）」がすっきりします。    
これでお試し版のforth-CF_MSXを作成することができますか？    
テストを行うために、"+"と"."（スタック上の値を表示してくれればどんなワードでも構いません）も追加願います。  
  
# ChatGPT:  
  
思考時間: 37s  
  
できます。ここでは **「おすすめの辞書フォーマット」＋「Token Threaded 実行系（JUMPのみ）」**で、まず **MSX-DOS .COM で動く“お試し版”**を作る形にします。  
  
* 実行系：token → ジャンプテーブル → jp (hl)（CALL/RETはVMでは使わない）    
* 即値：LIT token の直後に 16bit little-endian    
* テスト：2 3 + . を実行して終了    
* 辞書：ご指定の構造で 「+」「.」を登録（※このお試し版では“検索・解釈”までは未実装。辞書は形だけ先に用意）  
  
---  
  
## **1) 辞書エントリ構造（あなたの「すっきり」案）**  
  
1ワード：  
  
~~~
[ LINK(16) ][ FLAGS(8) ][ LEN(8) ][ NAME bytes ][ 0 ][ PAD ][ BODY_PTR(16) ][ BODY... ]  
~~~
  
* NAME...0 はNUL終端（ビット7終端は使わない）    
* PAD は BODY_PTR を 16bit 境界に合わせたい場合（今回は「0〜1バイトで調整」）    
* BODY_PTR は token列の先頭（このお試し版では「その単語を呼ぶ」機構はまだ使わない）  
  
---  
  
## **2) お試し版 forth-CF_MSX（Z80 / sjasmplus 用）**  
  
以下を forth_cf_msx_demo.asm として保存してください。  
  
~~~
; forth_cf_msx_demo.asm  
  
; MSX-DOS .COM demo: runs "2 3 + ." using token-threaded VM (JP-only)  
  
; assemble: sjasmplus --raw=forth_cf_msx_demo.com forth_cf_msx_demo.asm  
  
; run on MSX-DOS: forth_cf_msx_demo  
  
               org     0100h  
  
; -----------------------------  
  
; MSX-DOS BDOS call  
  
; CALL 0005h, C=function, DE args  
  
; function 2: output character in E  
  
; function 0: terminate  
  
; -----------------------------  
  
BDOS:           equ     0005h  
  
; -----------------------------  
  
; Token IDs (1-byte tokens)  
  
; -----------------------------  
  
T_LIT:          equ     0  
  
T_EXIT:         equ     1  
  
T_PLUS:         equ     2  
  
T_DOT:          equ     3  
  
T_HALT:         equ     4  
  
; -----------------------------  
  
; VM state  
  
; -----------------------------  
  
               ; Data Stack: use Z80 SP (fast)  
  
               ; Return Stack: memory area pointed by IX (grows down)  
  
               ; IP: DE points into token stream  
  
ds_area:        defs    256         ; 128 cells (16-bit) just for demo  
  
ds_top:                                 ; label after ds_area  
  
rp_area:        defs    128         ; 64 return addresses (16-bit)  
  
rp_top:                                 ; label after rp_area  
  
; -----------------------------  
  
; Jump table: token -> primitive entry  
  
; -----------------------------  
  
JT:  
  
               dw      PRIM_LIT  
  
               dw      PRIM_EXIT  
  
               dw      PRIM_PLUS  
  
               dw      PRIM_DOT  
  
               dw      PRIM_HALT  
  
; -----------------------------  
  
; Demo "program" as token stream:  
  
;   2 3 + .  
  
; -----------------------------  
  
thread_main:  
  
               db      T_LIT  
  
               dw      2  
  
               db      T_LIT  
  
               dw      3  
  
               db      T_PLUS  
  
               db      T_DOT  
  
               db      T_HALT  
  
; -----------------------------  
  
; Entry point  
  
; -----------------------------  
  
start:  
  
               ; init stacks  
  
               ld      sp, ds_top  
  
               ld      ix, rp_top  
  
               ; IP = thread_main  
  
               ld      de, thread_main  
  
               jp      NEXT  
  
; -----------------------------  
  
; NEXT: fetch token byte, dispatch via jump table, JP (HL)  
  
; DE = IP  
  
; -----------------------------  
  
NEXT:  
  
               ld      a, (de)     ; A = token  
  
               inc     de          ; IP++  
  
               ; HL = JT + token*2  
  
               ld      l, a  
  
               ld      h, 0  
  
               add     hl, hl      ; *2  
  
               ld      bc, JT  
  
               add     hl, bc  
  
               ; HL = *(JT + token*2)  
  
               ld      a, (hl)  
  
               inc     hl  
  
               ld      h, (hl)  
  
               ld      l, a  
  
               jp      (hl)        ; JP-only dispatch  
  
; -----------------------------  
  
; PRIM_LIT: ( -- n )  
  
; reads 16-bit literal from token stream, pushes to DS (SP)  
  
; -----------------------------  
  
PRIM_LIT:  
  
               ld      a, (de)  
  
               inc     de  
  
               ld      l, a  
  
               ld      a, (de)  
  
               inc     de  
  
               ld      h, a        ; HL = literal  
  
               push    hl  
  
               jp      NEXT  
  
; -----------------------------  
  
; PRIM_PLUS: ( n1 n2 -- n3 )  
  
; -----------------------------  
  
PRIM_PLUS:  
  
               pop     hl          ; n2  
  
               pop     bc          ; n1 (use BC as temp)  
  
               add     hl, bc  
  
               push    hl  
  
               jp      NEXT  
  
; -----------------------------  
  
; PRIM_DOT: ( n -- )  
  
; prints unsigned 16-bit in decimal + space  
  
; -----------------------------  
  
PRIM_DOT:  
  
               pop     hl  
  
               call    print_u16  
  
               ld      e, ' '  
  
               call    putch  
  
               jp      NEXT  
  
; -----------------------------  
  
; PRIM_EXIT: return from "call"  
  
; (in this demo, not used yet)  
  
; pops saved IP from return stack (IX), sets DE=IP, continues  
  
; RP grows down: store E then D (2 bytes)  
  
; -----------------------------  
  
PRIM_EXIT:  
  
               ld      e, (ix+0)  
  
               inc     ix  
  
               ld      d, (ix+0)  
  
               inc     ix  
  
               jp      NEXT  
  
; -----------------------------  
  
; PRIM_HALT: terminate to MSX-DOS  
  
; -----------------------------  
  
PRIM_HALT:  
  
               ld      c, 0  
  
               call    BDOS  
  
; -----------------------------  
  
; Console output helpers  
  
; -----------------------------  
  
putch:  
  
               ; E = char  
  
               push    af  
  
               push    bc  
  
               push    de  
  
               ld      c, 2  
  
               call    BDOS  
  
               pop     de  
  
               pop     bc  
  
               pop     af  
  
               ret  
  
; print_u16: HL = 0..65535, print decimal (no leading zeros), prints 0 correctly  
  
; division-free by subtracting place values  
  
print_u16:  
  
               push    af  
  
               push    bc  
  
               push    de  
  
               ld      de, 0       ; printed_flag in E (0/1), D unused  
  
               ; handle 0  
  
               ld      a, h  
  
               or      l  
  
               jr      nz, .p10000  
  
               ld      e, '0'  
  
               call    putch  
  
               jr      .done  
  
.p10000:  
  
               ld      bc, 10000  
  
               call    emit_digit  
  
               ld      bc, 1000  
  
               call    emit_digit  
  
               ld      bc, 100  
  
               call    emit_digit  
  
               ld      bc, 10  
  
               call    emit_digit  
  
               ; ones digit always printed  
  
               ld      bc, 1  
  
               call    emit_digit_force  
  
.done:  
  
               pop     de  
  
               pop     bc  
  
               pop     af  
  
               ret  
  
; emit_digit:  
  
; HL = value, BC = place (10000/1000/100/10)  
  
; uses: A digit count by repeated subtract (good enough for demo)  
  
; prints digit only if already printed something or digit != 0  
  
; printed_flag in E (0/1)  
  
emit_digit:  
  
               xor     a           ; A = digit  
  
.ed_loop:  
  
               or      a           ; keep flags sane  
  
               sbc     hl, bc  
  
               jr      c, .ed_restore  
  
               inc     a  
  
               jr      .ed_loop  
  
.ed_restore:  
  
               add     hl, bc      ; restore after overshoot  
  
               ; decide print  
  
               or      a  
  
               jr      nz, .ed_print  
  
               ld      a, e  
  
               or      a  
  
               ret     z           ; skip leading zeros  
  
               xor     a           ; digit=0 for printing  
  
.ed_print:  
  
               add     a, '0'  
  
               ld      e, 1        ; printed_flag=1  
  
               ld      e, e        ; (keep E=1)  
  
               ld      e, 1        ; ensure  
  
               ld      e, 1  
  
               ld      e, 1  
  
               ; putch expects char in E, so move A->E  
  
               ld      e, a  
  
               call    putch  
  
               ld      e, 1        ; restore printed_flag=1  
  
               ret  
  
; emit_digit_force:  
  
; always prints one digit for place=1  
  
emit_digit_force:  
  
               xor     a  
  
.ef_loop:  
  
               sbc     hl, bc  
  
               jr      c, .ef_restore  
  
               inc     a  
  
               jr      .ef_loop  
  
.ef_restore:  
  
               add     hl, bc  
  
               add     a, '0'  
  
               ld      e, a  
  
               call    putch  
  
               ret  
  
; -----------------------------  
  
; Dictionary (shape only for now)  
  
; [LINK][FLAGS][LEN][NAME...][0][PAD][BODY_PTR][BODY...]  
  
; In this demo, FIND/INTERPRET not implemented yet.  
  
; -----------------------------  
  
; flags (example)  
  
F_IMM:          equ     80h  
  
; helper: align to even address (for BODY_PTR word alignment)  
  
               macro   ALIGN_EVEN  
  
               if      ($ & 1)  
  
                       db 0  
  
               endif  
  
               endm  
  
; Word: "+"  
  
; BODY: single token T_PLUS then T_EXIT (so it can be "called" later)  
  
dict_plus:  
  
               dw      0           ; LINK (no previous for demo)  
  
               db      0           ; FLAGS  
  
               db      1           ; LEN  
  
               db      '+', 0  
  
               ALIGN_EVEN  
  
               dw      body_plus  
  
body_plus:  
  
               db      T_PLUS  
  
               db      T_EXIT  
  
; Word: "."  
  
dict_dot:  
  
               dw      dict_plus   ; LINK to "+"  
  
               db      0  
  
               db      1  
  
               db      '.', 0  
  
               ALIGN_EVEN  
  
               dw      body_dot  
  
body_dot:  
  
               db      T_DOT  
  
               db      T_EXIT  
  
LATEST:         dw      dict_dot  
  
               end     start  
~~~
  
### **注意（小さなバグ修正ポイント）**  
  
上の emit_digit は「printed_flag をEに入れる」実装を、分かりやすさ優先で雑に書いてあります。    
（Eをフラグ兼putch引数に使っているので、厳密には別レジスタに分けた方が綺麗です）  
  
もしこのまま動作が怪しかったら、. の表示を“16進”に簡略化すると一気に安定します。    
ただ、まずは「VMが回って 2 3 + . が出る」が目的なので、必要なら次の返信で「確実に動く版（10進/16進どちらでも）」に整えます。  
  
---  
  
## **3) これで何が確認できるか**  
  
* **token threaded 実行ループ（NEXT）**が jp (hl) で回る    
* LIT 直後の16bit即値が正しく push される    
* + が動く    
* . が値を出力できる    
* 辞書エントリが あなたの採用した構造で置ける  
  
---  
  
## **4) 次に足すと「Forthっぽく」なる最小2つ**  
  
このお試し版は「事前に用意した token列を実行するだけ」なので、次はどちらかを足すと一気にForthになります：  
  
1. FIND（入力トークン→辞書検索→BODY_PTR取得）    
2. CALL相当のtoken（例：T_CALL の直後に BODY_PTR を16bitで置き、RPにIP退避してジャンプ）  
  
これを入れると、body_plus / body_dot を本当に“呼べる”ようになります。  
  
---  
  
もしよければ次の返信で、あなたの辞書構造をそのまま使って  
  
* FIND（大文字小文字無視でも可）    
* T_CALL（CALL命令は使わず、VMとしては JUMPのみ）  
  
まで入れた「もう一歩Forth」版を出します。  
  
# あなた:  
  
forth_cf_msx_demo.asmをありがとうございます。内容をいじりながら確認しています。    
sjasmplus上でmakeすることで作成したCOMファイルをDSK化して、openMSXで実行しています。    
forth_cf_msx_demo.asmでは開始アドレスを0x100にして、startアドレスを0x2BBにしているので、openMSXのデバッガーでブレークポイントを0x2BBとしました。ですが、ブレークポイント近辺のメモリの16進ダンプを見ると、forth_cf_msx_demo.lstの値と異なっています。何が間違っているのでしょうか？  
  
# ChatGPT:  
  
思考時間: 55s  
  
起きていること、かなり「MSX-DOS + openMSXデバッガ」あるあるです。原因候補は大きく 2つあります。あなたの状況（.COMをDSK化→openMSX→デバッガでBPとダンプ比較）だと、ほぼこのどちらかです。  
  
---  
  
## **原因A：openMSXで見ている「0x0000〜0x3FFF」の中身が、タイミングによって別物**  
  
MSXは スロット/ページ切替があるので、同じCPUアドレス（例：0x02BB）でも、  
  
* MSX-BASIC中    
* DOS起動直後    
* .COM 実行中  
  
で そのアドレスに見えているメモリ（ROM/RAM/BDOS領域）が変わります。  
  
### **よくある症状**  
  
* DOS上で .COM を実行する前に bp 0x02bb を入れてダンプすると、そこはBIOS ROMや別領域で、.lst と一致しない。    
* .COM 実行後は一致するのに、実行前は一致しない。  
  
### **対処（確実）**  
  
1. MSX-DOSプロンプトで A> まで行く    
2. forth_cf_msx_demo を 実行開始した後（または実行直前）にデバッガで    
   * bp 0x02bb    
   * run / continue    
3. BPで止まった状態で、その時点のメモリをダンプして .lst と比較  
  
「BPを置くタイミング」がズレていると、ダンプが別物になります。  
  
---  
  
## **原因B：あなたの** .asm **は “エントリポイントが 0x0100 じゃない” 形になっている（これが混乱を生む）**  
  
org 0x0100 なのに、ソースの先頭に  
  
~~~
ds_area: defs 256  
  
rp_area: defs 128  
  
...  
  
start:  
  
 ...  
~~~
  
と 大きい defs を先に置いているので、.COM の先頭（0x0100）からしばらくは 0x00（NOP）だらけになります。  
  
* DOSは .COM を 0x0100から実行開始します    
* つまりCPUは 0x0100 から NOPをズルズル実行して、やっと 0x02BB の start に到達します  
  
これは「動くこともある」けど、デバッグではめちゃくちゃ混乱します。  
  
### **さらに厄介な点**  
  
openMSXのデバッガで  
  
* PCがどこを走っているか    
* どの時点でどの領域がRAM/ROMか  
  
が分かりづらくなり、結果として「.lst と一致しない」に見えやすいです。  
  
---  
  
## **いちばん簡単で確実な修正（おすすめ）**  
  
最初の0x0100に必ずコードを置き、即 jp start する方式に変えると、比較もBPも一気に素直になります。  
  
### **修正方針**  
  
* 0x0100 には必ず命令を置く（jp start）    
* defs のスタック領域は 末尾に移動（または org を変える）  
  
#### **例（冒頭だけこうする）**  
  
~~~
       org 0100h  
  
       jp  start  
  
; ここにコード類（JT、thread_main、NEXT、PRIM群など）  
  
start:  
  
       ld sp, ds_top  
  
       ld ix, rp_top  
  
       ld de, thread_main  
  
       jp NEXT  
  
; ファイル末尾にスタック領域  
  
ds_area: defs 256  
  
ds_top:  
  
rp_area: defs 128  
  
rp_top:  
~~~
  
これだと  
  
* .COM の実行開始地点は常に 0x0100 の jp start    
* start の番地も .lst と一致しやすい    
* 0x02BBみたいな「NOPスレッド」を踏まなくなる  
  
---  
  
## **まず確認してほしいチェックポイント（手元で一発判定）**  
  
1. .COM 実行中に BP で止めた状態で、PC（レジスタ表示）が本当に 0x02BB付近になっていますか？    
   * もし 0x0100〜0x02BBのどこかを走っているなら、NOP地帯の途中です。    
2. ダンプしているのは CPUアドレス空間の 0x02BB ですか？    
   * openMSXはスロット/セグメントを意識した表示もあるので、見方によっては「いま見えているページ」が違います。    
3. sjasmplus --raw=... で作った .COM と、DSKに入れたものが 同一ファイルですか？    
   * DSK作成ツールによっては別名/別パスの古いファイルを入れてしまう事故が地味に多いです。  
  
---  
  
## **こちらからの提案（次の一手）**  
  
もしよければ、あなたが使っている  
  
* sjasmplus の実行コマンド    
* DSK化の方法（どのツールでどう入れたか）    
* openMSX上での実行方法（DOS1/DOS2、起動手順）    
* .lst の start 行の前後 10行くらい    
* openMSXデバッガで止めた時の PC値 と、0x02BB付近のダンプ数十バイト  
  
このうち 2〜3個を貼ってくれれば、どっち（原因A/B）か即断して、あなたの環境に合わせて「ここをこう直せば一致する」をピンポイントで言えます。  
  
ただ、今の話だけで「最も効く改善」は **原因Bの修正（0x0100に jp start、defsを末尾）**なので、まずそれを入れるのが一番早いです。  
  
# あなた:  
  
MSXのメモリマップ上でCOMファイルへロードして実行するのにいい場所は決まっていますか？  
  
# ChatGPT:  
  
思考時間: 8s  
  
はい、基本はほぼ「決まっています」。MSX-DOS の .COM は原則として  
  
* ロード先：0x0100    
* 実行開始：0x0100  
  
です。なので「COMをロードして実行するのに良い場所」を自分で選ぶ、というより MSX-DOS がそういう形式として固定しています。  
  
---  
  
## .COM **が 0x0100 固定な理由**  
  
MSX-DOS は CP/M と同系統の作法で、  
  
* 0x0000〜0x00FF：DOS が用意する領域（コマンドライン、FCB、DMA など）    
* 0x0100〜：Transient Program Area（TPA）    
  → ここに .COM を丸ごと読み込んで、そのまま走らせる  
  
という前提で動きます。  
  
あなたが org 0100h にしているのは正しいです。  
  
---  
  
## **「好きな番地に置きたい」ときは？**  
  
.COM では基本できません（DOSが0x0100に置く前提だから）。やりたくなったら選択肢はこうです：  
  
1. .COM の中で自分で別領域にコピーしてそこへ飛ぶ    
   * いわゆる「自己再配置（relocation）」    
   * ただしMSXはスロット/メモリマッパ絡みがあるので、初心者向けではありません    
2. MSX-DOS2 のメモリマッパ機能や、システムコールでメモリ確保して使う    
   * “大きい領域” が必要になったらこの方向    
   * ただしDOS1ではできることが限られます    
3. .COM をやめて .BIN（BLOAD/RUN）やROM形式にする    
   * これは「DOS実行」ではなくMSX側の別のロード機構になります  
  
---  
  
## **実務的に「いい配置」はどうする？**  
  
.COM の場合は実務的にはこう決めるのが鉄板です：  
  
* コードは org 0100h から始める    
* 先頭に必ず jp start を置く（前に defs を置かない）    
* その後ろにテーブルやスレッド    
* 末尾にスタック/バッファ（defs）を置く  
  
こうしておくと  
  
* PC=0x0100からすぐ意味のあるコードに入る    
* .lst とデバッガのメモリダンプが一致しやすい    
* 「いつの間にかNOP地帯を走ってる」事故が消える  
  
---  
  
## **どこまで使っていいか（サイズ制限）**  
  
.COM は TPAの範囲内で動きます。TPAの上限は  
  
* DOS1 / DOS2    
* 常駐物（ドライバ、RAMディスク等）    
* メモリマッパ状況  
  
で変わるので、上限アドレスを固定値で決め打ちしないのが安全です。  
  
まずは今回みたいな小さいVMなら、0x0100基点で「全部 .COM の中」に収めるのが一番確実です。  
  
---  
  
### **まとめ**  
  
* MSX-DOSの .COM は 0x0100 にロードして 0x0100 から実行が基本で、場所は実質固定    
* “いい場所”を選ぶより、0x0100起点で素直に配置するのが正解    
* 先頭に jp start、defs は末尾へ、がデバッグしやすくおすすめ  
  
必要なら、あなたの forth_cf_msx_demo.asm を「0x0100に jp start 」「defs を末尾」の形に整えた版をこちらで提示します。  
  
# あなた:  
  
forth_cf_msx_demo.asmを2か所修正して、正しく動作することを確認しました。    
１）ORG 0x100の後にJP startを挿入した。    
２）最初に書かれていたスタック領域の定義を一番最後にした。  
  
ここには書かれていないが、デバッグ中に画面に情報を表示させたかったのにうまくいかなかった。理由がわかった時に、「これは盲点だった」と驚いた。   
  
・B-DOSで文字列出力をする場合は、文字列の最後に”＄”をつけること（0x00ではない）。  
  
＜＜完成版を添付します＞＞    
~~~
; forth_cf_msx_demo.asm    
; MSX-DOS .COM demo: runs "2 3 + ." using token-threaded VM (JP-only)    
; assemble: sjasmplus --raw=forth_cf_msx_demo.com forth_cf_msx_demo.asm    
; run on MSX-DOS: forth_cf_msx_demo  
  
                org     0100h    
                    
               jp  start   
  
; -----------------------------    
; MSX-DOS BDOS call    
; CALL 0005h, C=function, DE args    
; function 9: output string    
; function 2: output character in E    
; function 0: terminate    
; -----------------------------    
BDOS:           equ     0005h  
  
; -----------------------------    
; Token IDs (1-byte tokens)    
; -----------------------------    
T_LIT:          equ     0    
T_EXIT:         equ     1    
T_PLUS:         equ     2    
T_DOT:          equ     3    
T_HALT:         equ     4  
  
; -----------------------------    
; Entry point    
; -----------------------------    
start:    
                ; init stacks    
                ld      sp, ds_top    
                ld      ix, rp_top  
  
                ; IP = thread_main    
                ld      de, thread_main    
                jp      NEXT  
  
; -----------------------------    
; NEXT: fetch token byte, dispatch via jump table, JP (HL)    
; DE = IP    
; -----------------------------    
NEXT:    
                ld      a, (de)     ; A = token    
                inc     de          ; IP++  
  
                ; HL = JT + token*2    
                ld      l, a    
                ld      h, 0    
                add     hl, hl      ; *2    
                ld      bc, JT    
                add     hl, bc  
  
                ; HL = *(JT + token*2)    
                ld      a, (hl)    
                inc     hl    
                ld      h, (hl)    
                ld      l, a  
  
                jp      (hl)        ; JP-only dispatch  
  
; -----------------------------    
; PRIM_LIT: ( -- n )    
; reads 16-bit literal from token stream, pushes to DS (SP)    
; -----------------------------    
PRIM_LIT:    
                ld      a, (de)    
                inc     de    
                ld      l, a    
                ld      a, (de)    
                inc     de    
                ld      h, a        ; HL = literal    
                push    hl    
                jp      NEXT  
  
; -----------------------------    
; PRIM_PLUS: ( n1 n2 -- n3 )    
; -----------------------------    
PRIM_PLUS:    
                pop     hl          ; n2    
                pop     bc          ; n1 (use BC as temp)    
                add     hl, bc    
                push    hl    
                jp      NEXT  
  
; -----------------------------    
; PRIM_DOT: ( n -- )    
; prints unsigned 16-bit in decimal + space    
; -----------------------------    
PRIM_DOT:    
                pop     hl    
                call    print_u16    
                ld      e, ' '    
                call    putch    
                jp      NEXT  
  
; -----------------------------    
; PRIM_EXIT: return from "call"    
; (in this demo, not used yet)    
; pops saved IP from return stack (IX), sets DE=IP, continues    
; RP grows down: store E then D (2 bytes)    
; -----------------------------    
PRIM_EXIT:    
                ld      e, (ix+0)    
                inc     ix    
                ld      d, (ix+0)    
                inc     ix    
                jp      NEXT  
  
; -----------------------------    
; PRIM_HALT: terminate to MSX-DOS    
; -----------------------------    
PRIM_HALT:    
                ld      c, 0    
                call    BDOS  
  
; -----------------------------    
; Console output helpers    
; -----------------------------    
putch:    
                ; E = char    
                push    af    
                push    bc    
                push    de    
                ld      c, 2    
                call    BDOS    
                pop     de    
                pop     bc    
                pop     af    
                ret  
  
; print_u16: HL = 0..65535, print decimal (no leading zeros), prints 0 correctly    
; division-free by subtracting place values    
print_u16:    
                push    af    
                push    bc    
                push    de  
  
                ld      de, 0       ; printed_flag in E (0/1), D unused    
                ; handle 0    
                ld      a, h    
                or      l    
                jr      nz, .p10000    
                ld      e, '0'    
                call    putch    
                jr      .done  
  
.p10000:    
                ld      bc, 10000    
                call    emit_digit    
                ld      bc, 1000    
                call    emit_digit    
                ld      bc, 100    
                call    emit_digit    
                ld      bc, 10    
                call    emit_digit    
                ; ones digit always printed    
                ld      bc, 1    
                call    emit_digit_force  
  
.done:    
                pop     de    
                pop     bc    
                pop     af    
                ret  
  
; emit_digit:    
; HL = value, BC = place (10000/1000/100/10)    
; uses: A digit count by repeated subtract (good enough for demo)    
; prints digit only if already printed something or digit != 0    
; printed_flag in E (0/1)    
emit_digit:    
                xor     a           ; A = digit    
.ed_loop:    
                or      a           ; keep flags sane    
                sbc     hl, bc    
                jr      c, .ed_restore    
                inc     a    
                jr      .ed_loop    
.ed_restore:    
                add     hl, bc      ; restore after overshoot    
                ; decide print    
                or      a    
                jr      nz, .ed_print    
                ld      a, e    
                or      a    
                ret     z           ; skip leading zeros    
                xor     a           ; digit=0 for printing    
.ed_print:    
                add     a, '0'    
                ld      e, 1        ; printed_flag=1    
                ld      e, e        ; (keep E=1)    
                ld      e, 1        ; ensure    
                ld      e, 1    
                ld      e, 1    
                ; putch expects char in E, so move A->E    
                ld      e, a    
                call    putch    
                ld      e, 1        ; restore printed_flag=1    
                ret  
  
; emit_digit_force:    
; always prints one digit for place=1    
emit_digit_force:    
                xor     a    
.ef_loop:    
                sbc     hl, bc    
                jr      c, .ef_restore    
                inc     a    
                jr      .ef_loop    
.ef_restore:    
                add     hl, bc    
                add     a, '0'    
                ld      e, a    
                call    putch    
                ret  
  
; -----------------------------    
; Dictionary (shape only for now)    
; [LINK][FLAGS][LEN][NAME...][0][PAD][BODY_PTR][BODY...]    
; In this demo, FIND/INTERPRET not implemented yet.    
; -----------------------------  
  
; flags (example)    
F_IMM:          equ     80h  
  
; helper: align to even address (for BODY_PTR word alignment)    
                macro   ALIGN_EVEN    
                if      ($ & 1)    
                        db 0    
                endif    
                endm  
  
; Word: "+"    
; BODY: single token T_PLUS then T_EXIT (so it can be "called" later)    
dict_plus:    
                dw      0           ; LINK (no previous for demo)    
                db      0           ; FLAGS    
                db      1           ; LEN    
                db      '+', 0    
                ALIGN_EVEN    
                dw      body_plus    
body_plus:    
                db      T_PLUS    
                db      T_EXIT  
  
; Word: "."    
dict_dot:    
                dw      dict_plus   ; LINK to "+"    
                db      0    
                db      1    
                db      '.', 0    
                ALIGN_EVEN    
                dw      body_dot    
body_dot:    
                db      T_DOT    
                db      T_EXIT  
  
LATEST:         dw      dict_dot  
  
; -----------------------------    
; Jump table: token -> primitive entry    
; -----------------------------    
JT:    
                dw      PRIM_LIT      ; 0    
                dw      PRIM_EXIT     ; 1    
                dw      PRIM_PLUS     ; 2    
                dw      PRIM_DOT      ; 3    
                dw      PRIM_HALT     ; 4  
  
; -----------------------------    
; Demo "program" as token stream:    
;   2 3 + .    
; -----------------------------    
thread_main:    
                db      T_LIT    
                dw      2    
                db      T_LIT    
                dw      3    
                db      T_PLUS    
                db      T_DOT    
                db      T_HALT  
  
; -----------------------------    
; VM state    
; -----------------------------    
                ; Data Stack: use Z80 SP (fast)    
                ; Return Stack: memory area pointed by IX (grows down)    
                ; IP: DE points into token stream  
  
ds_area:        defs    256         ; 128 cells (16-bit) just for demo    
ds_top:                                 ; label after ds_area  
  
rp_area:        defs    128         ; 64 return addresses (16-bit)    
rp_top:                                 ; label after rp_area  
  
                end     start  
~~~
  
# ChatGPT:  
  
修正点も「盲点だった」も、どれも筋が通ってます。特に  
  
* org 0100h の直後に jp start    
* defs を末尾へ  
  
は MSX-DOS .COM では鉄板で、デバッガ上の番地ズレや「NOP地帯を実行してた」問題が一気に消えます。  
  
そして BDOS関数9の $終端、これは本当に引っかかりがちですね（CP/M由来の仕様）。気づけたのは大きいです。  
  
---  
  
## **いまの完成版で “次に気になる” であろう2点（軽く指摘）**  
  
### **1)** print_u16 **の** emit_digit **が、まだちょっと危ない**  
  
元の雛形の時点でも「Eをフラグ兼putch引数にしていて雑」と書いた部分が残っています。  
  
あなたの版でも：  
  
* print_u16 で de=0（printed_flag を E で表現）    
* emit_digit 内で putch に渡すために e=a にしてしまい    
* その後 e=1 に戻す  
  
という操作が入っています。  
  
偶然うまく見えることはあるのですが、桁ごとにフラグの意味がブレやすいです。    
（デバッグ用なら十分、という割り切りでもOK）  
  
簡単に安全化するなら：  
  
* printed_flag を B か D に持つ（putchが壊さないレジスタにする）    
* putch に渡す文字はE固定のままでもよい  
  
このあたり、次の段階で「辞書検索のデバッグ表示」を多用するようになると、必ず効いてきます。  
  
---  
  
### **2) 辞書（dict_*) と実行系がまだ結びついていない**  
  
今は thread_main が固定で、  
  
* dict_plus / dict_dot は “形だけ”  
  
ですよね。次のステップはここを繋げるのが自然です。  
  
---  
  
## **次に進めるなら：この完成版を “Forthっぽく” する最小追加**  
  
あなたが目標としている「CF方式（token VM）＋一体辞書（BODY_PTR）」に一番近い、最小の追加はこれです：  
  
### **追加A：**FIND**（入力トークン→辞書→BODY_PTR）**  
  
* LATEST からLINKを辿る    
* LEN一致→NAME比較（NUL終端があるので比較しやすい）    
* 一致したら BODY_PTR を取り出す  
  
### **追加B：**T_CALL **token（CALL命令は使わない）**  
  
* token列に T_CALL を追加    
* T_CALL の直後に 16bitのBODY_PTR を置く    
* T_CALL 実装が：    
  1. 現在のIP(DE)をRP(IX)へpush    
  2. DE=BODY_PTR に更新    
  3. jp NEXT    
* ワードのBODY末尾は T_EXIT（既に入れてあるので相性良い）  
  
これで、たとえば  
  
~~~
[ LIT 2 ][ LIT 3 ][ CALL body_plus ][ CALL body_dot ][ HALT ]  
~~~
  
みたいに「辞書ワードを呼ぶ」流れが完成します。  
  
---  
  
## **BDOSの** $ **終端、整理しておくと今後役立ちます**  
  
* BDOS function 9（文字列出力）は DE が文字列先頭、終端は $    
* NUL終端（0）じゃない    
* 改行も自動ではないので、必要なら CR(0x0D), LF(0x0A) を自分で入れる  
  
なので今後、デバッグ文字列は例えば：  
  
~~~
msg_hello: db "IP=", '$'  
~~~
  
のようにするのが安全です。  
  
---  
  
## **ここから先、こちらで“本当に動く次版”を出せます**  
  
あなたの完成版をベースに、次のどれか（または全部）を入れた forth_cf_msx_step2.asm をこちらで提示できます：  
  
1. print_u16 を堅牢化（フラグ/レジスタ整理）    
2. BDOS 9 用の puts_dollar を追加（デバッグが楽になる）    
3. T_CALL を追加して、dict_plus/dict_dot を本当に呼べるようにする    
4. さらに FIND を追加して「文字列から辞書検索」まで行く  
  
あなたはすでに openMSX で動作確認までできているので、次は (3) → (4) の順が気持ちよく進むと思います。  
  
「次は T_CALL で辞書ワードを呼ぶ」まで進めますか？それとも先に「FIND を作って入力から実行」まで一気に行きますか？  
  
[(次へ進む)](./CF-MSX_07_a.md)      
[(README.mdへ戻る)](../README.md)      
