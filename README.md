## Basic Concept
-- __Inspired by Tatsuyuki Inoue__ --

    ( x - α_1 )* ( x - α_3 )* ( x - α_5 )* ( x - α_7 )*
	
	( x - β_1 )* ( x - β_3 )* ...

    α_1 〜 α_4： ±AABBCCDDEEZ // 平文
    | AA - EE: 平文の分割されたデータ(長さ：param_chars)
    | Z: 順番に 0 〜 7 // 復号化時に平文を再構築するため
		
    β_1 〜 β_(NUM_OF_KEYS*2): ±AABBCCDDEEZ // 鍵
    | AA...: 鍵のSHA512の上位(PARAM_CHARS)*(NUM_OF_KEYS*2)バイト  
    |        長さが足りなくなったらhashの更にhashをとってつなげる。
	|		 平文の方と同じ長さ
    | Z: 順番に 0 ~ 7
	
といった感じで平文と鍵を表す整数を根にもつ式を展開し、その係数のみを取ることで暗号化するブロック暗号。  
効率的な因数分解アルゴリズムは色々あるが、5次以上の方程式の解の公式は存在しないことを利用する。  
平文のみの式は4次式になるようにし、鍵を組み合わせることで5次式以上にする。  
復号化するときは鍵が根になることを利用して4次式に次数下げしてから解く。  

数式処理には[SymPy](http://sympy.org/en/index.html)を使ってるので、動作にはこいつのインストールが必要。  

## Strong Points
以下の2つの仕組みによって成り立っている。  

1. **５次以上の方程式の解の公式が存在しない、つまり因数分解に時間がかかることを利用。**  

2. **強引に因数分解されても、鍵と平文の区別がつかない** (下位8bitが平文と鍵で同じものができるため)  
   得られた根から、平文の断片4つの考え得る組み合わせを全て試すこともできるが、  
   鍵の個数が多い(=`num_of_keys`の値が大きい)ほど解読は困難になると言える。  
   その組み合わせの数は以下のように計算できる。  
   
   ```python
   # num_of_keys をnで表すとすると
   num_of_combinations = (n / 4 + 2)**(n % 4) * (n / 4 + 1)**(4-(n % 4))
   ```  
   
   ![組み合わせの数](https://github.com/pheehs/QuinticEncrypt/raw/master/growing_comb2.png "組み合わせの数")  
   **要するに少し鍵の個数を増やせばめちゃくちゃ解読しづらくなる。**

## Weak Points
1. _強引に因数分解される可能性が十分ある。_  
   -> どれぐらい次数を上げれば因数分解に非現実的な時間がかかるようになるか要検証  
      (計算量的安全性の確保)
  
2. _特殊な状況だと解読可能_  
   因数分解できた場合でも解読は難しいが、  
   **同じ平文あるいは同じ鍵に対する暗号文2つ以上を知られる場合**は、  
   鍵と平文の両方を完全に解読できる。(crack()に実装)  
   
   -> 対策思いつかない

同じ鍵で暗号化した暗号文２つからそれぞれの平文を解読するのにかかる時間を計測し、  
鍵を使って通常通り復号化した時にかかる時間で割ったものをグラフにするとこうなる。  
![解読時間](https://github.com/pheehs/QuinticEncrypt/raw/master/crack_time_rate_plot.png "解読時間")  
実質的には因数分解にかかる時間を測っているのとほぼ同じと思われる。
グラフにある範囲程度だと解読は容易。

## Bugs and Points to be improved
1. _暗号化後のデータサイズが大きい_  
   -> 暗号後、自動的に何らかのアルゴリズム(zlibの予定)で圧縮する(オプションで追加)

## Benchmark
`"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz This is plain data."`(=72bytes)を平文として、`param_chars`と`num_of_keys`を変化させて暗号化・復号化にかかる時間を計測(benchmark())、gnuplotでグラフにしてみた。  
![ベンチマーク結果](https://github.com/pheehs/QuinticEncrypt/raw/master/benchmark_plot2.png "ベンチマーク結果")  
`num_of_keys`は小さければ小さいほど、実行時間は若干短くなり、  
`param_chars`も基本的には大きいほど実行時間が短くなるが、  
特定の`param_chars`で突然実行時間が長くなる。（原因不明。）

## Structure of Encrypted file

    |length|param|                          equation_1                      ||   equation_2 ..
    |of org|chars|          header              ||          body            ||
    |data  |     |dim|nd-len| ... |2d-len|1d-len||nd-int| ... |2d-int|1d-int||
    |4bytes|  2  | 2 |  4   |     |  4   |  4   ||  ?   |     |  ?   |   ?  ||

## Notes
* もともと5次方程式を利用するつもりだったが、拡張して次数が自由に変えられるようになったので**Quintic**Encryptじゃない。
* RSA暗号は整数の因数分解(素因数分解)の困難性を利用するのに対し、これは整式の因数分解の困難性を利用。
* この暗号方式の解と平文の見分けがつかないという整式の性質を、整数の場合も使えないか？

a polynomial expression 多項式  
high-powered equation 高次方程式  

5次方程式  
quintic equation  

↓_次数q(ﾟдﾟ )↓sage↓_

4次方程式  
quartic equation  

1. 複二次式の場合
  二次式と同様に解ける
2. それ以外の場合
  x^3の項を消して、フェラーリの方法orデカルトの方法orオイラーの方法で3次式と2次式に
  参考:[wikipedia](http://ja.wikipedia.org/wiki/4%E6%AC%A1%E6%96%B9%E7%A8%8B%E5%BC%8F)など

↓_次数q(ﾟдﾟ )↓sage↓_

3次方程式  
cubic equation  

2次方程式  
quadratic equation  
