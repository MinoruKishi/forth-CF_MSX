# GitHub MinoruKishi/forth-CF_MSX   
  
## CCurl/CFを初めて読んだ人がChatGPTの力を借りてのMSX0向けFORTH改造の顛末記〜０８  
  
# あなた:  
  
以下のDE_Tとcellを教えてください。    
~~~
typedef struct { cell xt; byte flags, len; char name[NAME_MAX+1]; } DE_T;  
~~~
  
# ChatGPT:  
  
とても大事な部分ですね。    
これは 辞書エントリ（Dictionary Entry）構造体 です。    
CF / Forth 系 VM の「ワード1個分のヘッダ」を表しています。  
  
---  
  
# **全体構造**  
  
~~~
typedef struct {  
  
   cell xt;  
  
   byte flags, len;  
  
   char name[NAME_MAX+1];  
  
} DE_T;  
~~~
  
つまり：  
  
| フィールド | 役割 |  
| ----- | ----- |  
| xt | 実行アドレス |  
| flags | 属性ビット |  
| len | ワード名の長さ |  
| name | ワード名 |  
  
---  
  
# **1️⃣ cell とは？**  
  
cell は Forth の基本単位です。  
  
### **Forthの「セル」**  
  
Forth では：  
  
  `「1セル = VMの基本整数サイズ」`  
  
環境によって違います。  
  
| 環境 | cellサイズ |  
| ----- | ----- |  
| 16bit Forth | 16bit |  
| 32bit Forth | 32bit |  
| 64bit Forth | 64bit |  
  
CFの場合、たとえば：  
  
  `typedef intptr_t cell;`  
  
のように定義されている可能性があります。  
  
つまり：  
  
  `cell = VMの整数型`  
  
---  
  
# **2️⃣ DE_T とは？**  
  
DE_T は  
  
  `Dictionary Entry Type`  
  
の略です。  
  
つまり  
  
  `ワード1個分の辞書ヘッダ`  
  
です。  
  
---  
  
# **3️⃣ 各メンバを詳しく**  
  
## **(1)** cell xt;  
  
### **xt = Execution Token**  
  
Forthで言う：  
  
  `' dup`  
  
で得られる値。  
  
つまり：  
  
このワードを実行するためのアドレス  
  
CFでは通常：  
  
* PRIMS番号    
* コード配列の位置    
* 関数ポインタ  
  
などが入ります。  
  
---  
  
## **(2)** byte flags  
  
ワードの属性です。  
  
典型例：  
  
| フラグ | 意味 |  
| ----- | ----- |  
| IMMEDIATE | コンパイル中でも即実行 |  
| HIDDEN | 検索対象外 |  
| COMPILE_ONLY | 実行禁止 |  
  
CF内部でビット管理されています。  
  
---  
  
## **(3)** byte len  
  
ワード名の長さ。  
  
Forthは文字列終端に依存せず、  
  
  `len + name`  
  
で高速比較することが多いです。  
  
---  
  
## **(4)** char name[NAME_MAX+1]  
  
ワード名。  
  
+1 は  
  
  `Cの文字列終端 '\0' 用`  
  
です。  
  
例：  
  
  `dup`  
  
なら：  
  
~~~
len = 3  
  
name = {'d','u','p','\0'}   --> CF-MSXではMSX-DOS内部では以下のようになりそうです。  
  name_CF_MSX = {'d','u','p','$','\0'}  
~~~
---  
  
# **4️⃣ メモリ上のイメージ**  
  
~~~code
+-------------------+  
|　xt　　　　　　　　　| ← 実行アドレス  
+-------------------+  
| flags | len       |  
+-------------------+  
| name[0]           |  
| name[1]           |  
| name[2]           |  
| ...               |  
+-------------------+    
~~~
---  
  
# **5️⃣ これは「辞書リンク」を持たない？**  
  
伝統的Forthでは：  
  
+ link  
  
+ flags+len  
  
+ name  
  
+ code field  
  
という構造ですが、  
  
CFでは：  
  
  `辞書を配列管理している`  
  
可能性が高いです。  
  
つまり：  
  
  `DE_T dict[MAX_WORDS];`  
  
のように。  
  
---  
  
# **6️⃣ CF的に重要なポイント**  
  
あなたが以前整理していた通り：  
  
* Cコアは固定    
* .fthで追加されるのはワード定義  
  
この DE_T は  
  
  `Cコアが管理するワードの登録テーブル`  
  
です。  
  
---  
  
# **7️⃣ MSX/Z80移植で重要になる点**  
  
MSX0へ移植する場合：  
  
問題になるのは：  
  
* cell サイズ    
* xt の意味    
* 辞書の配置場所  
  
特にZ80では：  
  
  `cell = 16bit`  
  
にする必要があります。  
  
---  
  
# **8️⃣ まとめ**  
  
### **DE_T = ワード1個分の辞書エントリ**  
  
~~~
xt     → 実行アドレス  
  
flags  → 属性  
  
len    → 名前長  
  
name   → ワード名  
~~~
  
### **cell = Forthの基本整数型**  
  
---  
  
もしよければ次に：  
  
* xtはPRIMS番号か？    
* colon定義はどう入る？    
* MSX版ではこの構造をどう置くべきか？  
  
まで一気に掘りますか？  
  
あなたのCF→MSX移植理解に直結する核心部分です。  
  
[(次へ進む)](./CF-MSX_09_a.md)      
[(README.mdへ戻る)](../README.md)      
