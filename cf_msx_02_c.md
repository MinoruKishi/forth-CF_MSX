# GitHub MinoruKishi/forth-CF_MSX   
  
## CCurl/CFを初めて読んだ人がChatGPTの力を借りてのMSX0向けFORTH改造の顛末記  
  
#   
  
# あなた:  
  
sjasmplusでマクロ内でIF分を使うときに、マクロの変数名とHLなどのレジスタ名を比較したい  
  
# ChatGPT:  
  
sjasmplus で マクロ引数（reg）と Z80 レジスタ名（HL / BC / DE など）を IF で比較したい場合、    
結論から言うと：  
  
IF reg=="HL" のような比較はできません    
→ sjasmplus の IF は「文字列比較」ではなく「アセンブル時の式評価」だからです。  
  
そのため、    
reg が HL のとき  
  
'IF reg=="HL"'  
  
は  
  
 'IF HL=="HL"'  
  
に展開され、HL は未定義シンボル → warning / error になります。  
  
---  
  
## **正解の考え方（重要）**  
  
sjasmplus のマクロ引数は    
「文字列」ではなく「トークン置換」 です。  
  
したがって、    
👉 「値として比較」するのではなく    
👉 「識別子として分岐」させる設計にする  
  
のが正解です。  
  
---  
  
## **解決策①（推奨）：レジスタごとにマクロを分ける**  
  
### **呼び出し側**  
  
I_JMP_IND_HL  
  
I_JMP_IND_BC  
  
I_JMP_IND_DE  
  
### **定義側**  
  
MACRO I_JMP_IND_HL  
  
   PUSH HL  
  
   JP (HL)  
  
ENDM  
  
MACRO I_JMP_IND_BC  
  
   PUSH BC  
  
   JP (BC)  
  
ENDM  
  
MACRO I_JMP_IND_DE  
  
   PUSH DE  
  
   JP (DE)  
  
ENDM  
  
✔ 最も安全    
✔ sjasmplus の思想に合っている    
✔ エラー原因が明確  
  
---  
  
## **解決策②：レジスタ名を「数値」に変換して比較する（高度）**  
  
「どうしても IF で分岐したい」場合は    
文字列比較を捨てて、数値で判定します。  
  
### **1️⃣ 呼び出し側（数値を渡す）**  
  
I_JMP_IND 0   ; HL  
  
I_JMP_IND 1   ; BC  
  
I_JMP_IND 2   ; DE  
  
### **2️⃣ マクロ定義**  
  
MACRO I_JMP_IND regid  
  
   IF regid == 0  
  
       PUSH HL  
  
       JP (HL)  
  
   ELSEIF regid == 1  
  
       PUSH BC  
  
       JP (BC)  
  
   ELSEIF regid == 2  
  
       PUSH DE  
  
       JP (DE)  
  
   ELSE  
  
       .ERROR "invalid register id"  
  
   ENDIF  
  
ENDM  
  
✔ IF が正しく動作    
✔ sjasmplus 的には「正統派」  
  
---  
  
## **解決策③（裏技・非推奨）：未定義シンボル検出**  
  
MACRO I_JMP_IND reg  
  
   IFDEF HL  
  
       ; これは意味を持たない  
  
   ENDIF  
  
ENDM  
  
❌ 使い物になりません    
sjasmplus ではマクロ引数は IFDEF 判定対象になりません。  
  
---  
  
## **なぜ sjasmplus では文字列比較ができないのか？**  
  
sjasmplus の IF は：  
  
* C プリプロセッサではない    
* 「展開後の式」を評価する    
* "HL" のような文字列リテラルを扱えない  
  
つまり：  
  
IF reg=="HL"  
  
は 思想的に不可能 です。  
  
---  
  
## **おすすめ設計（Z80 + FORTH 系なら）**  
  
あなたの流れ（8086 → Z80 / FORTH VM 移植）を考えると：  
  
### **👍 ベストプラクティス**  
  
* レジスタ別マクロ    
* もしくは regid 数値化    
* 「1 命令 = 1 マクロ」  
  
これは colorForth / VM 実装とも非常に相性が良いです。  
  
---  
  
## **まとめ**  
  
| 方法 | 可否 | 推奨度 |  
| ----- | ----- | ----- |  
| IF reg=="HL" | ❌ 不可 | ✖ |  
| マクロ分割 | ✅ | ⭐⭐⭐ |  
| 数値 ID 分岐 | ✅ | ⭐⭐ |  
| IFDEF 文字列判定 | ❌ | ✖ |  
  
---  
  
必要なら次に：  
  
* 8086 マクロ → Z80 マクロの安全な自動変換設計    
* sjasmplus 向け FORTH VM マクロ規約    
* 「わざとエラーを出す」ための sjasmplus 流 .ERROR 設計  
  
も一緒に整理できますよ。  
  
# あなた:  
  
たとえば、以下の場合はどうでしょうか？  
  
~~~    
R_AX EQU 0    
R_BX EQU 1    
R_DX EQU 2  
  
I_JMP_IND R_AX   ; HL     
I_JMP_IND R_BX   ; BC     
I_JMP_IND R_DX   ; DE  
  
MACRO I_JMP_IND regid    
    IF regid == 0    
        PUSH HL    
        JP (HL)    
    ELSEIF regid == 1    
        PUSH BC    
        JP (BC)    
    ELSEIF regid == 2    
        PUSH DE    
        JP (DE)    
    ELSE    
        .ERROR "invalid register id"    
    ENDIF    
ENDM    
~~~   
  
# ChatGPT:  
  
その書き方、sjasmplus的にかなり良いです。    
ポイントは「マクロ引数 regid が *数値として評価できる*（EQU 定数に展開される）」ことなので、  
  
* R_AX EQU 0 → I_JMP_IND R_AX → regid == 0 が真    
* R_BX EQU 1 → regid == 1 が真    
* R_DX EQU 2 → regid == 2 が真  
  
…という形で 期待通りに動きます。  
  
---  
  
## **ただし、より安全にする小改善**  
  
### **1) “0/1/2” を直書きしない（定数で比較）**  
  
あとで番号を増やすときに事故りやすいので、比較も定数で書くのが安全です。  
  
R_AX EQU 0  
  
R_BX EQU 1  
  
R_DX EQU 2  
  
MACRO I_JMP_IND regid  
  
   IF regid == R_AX  
  
       PUSH HL  
  
       JP (HL)  
  
   ELSEIF regid == R_BX  
  
       PUSH BC  
  
       JP (BC)  
  
   ELSEIF regid == R_DX  
  
       PUSH DE  
  
       JP (DE)  
  
   ELSE  
  
       .ERROR "invalid regid for I_JMP_IND"  
  
   ENDIF  
  
ENDM  
  
### **2) “コメント”を正しい対応関係にしておく**  
  
例では R_AX ; HL になってますが、読み手が混乱しやすいので、対応表をコメントで固定しておくのがおすすめです。  
  
; regid mapping:  
  
;   R_AX -> HL  
  
;   R_BX -> BC  
  
;   R_DX -> DE    
---  
  
## .ERROR **が効くか？**  
  
sjasmplus にはエラーメッセージを出す系の疑似命令があるので、.ERROR 相当が使えればOKです。    
（もし環境で .ERROR が通らない場合は、sjasmplus が認識する別の命令名に変える必要があります。そのときは“意図的にエラーを出す”代替も出せます。）  
  
---  
  
## **さらに拡張するなら（例：SP / IX / IY も）**  
  
JP (IX) みたいに Z80が許す間接形式だけ追加していけば、分岐は同じ設計で増やせます。  
  
---  
  
### **結論**  
  
あなたの例は 「文字列比較できない問題」を完全に回避できていて正しいです。    
この路線で、8086側の AX/BX/DX を「仮想レジスタID」にして Z80 の HL/BC/DE に割り当てるのは、移植作業としても扱いやすいです。  
  
もし「PUSH/POP の対応も同じ regid で統一したい」みたいな狙いがあるなら、その枠組み（I_PUSH / I_POP / I_MOV 等）も同じ表で揃える案を作れます。  
  
[(次へ進む)](./CF-MSX_03_a.md)      
[(README.mdへ戻る)](../README.md)      
