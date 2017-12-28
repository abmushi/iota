IOTA:Signature And Validation

# Curl and Kerl
　link [here](https://github.com/iotaledger/kerl)。

# Address generation
 First of all, create **Private Key** from **Seed**.
 
 ```js:
// length = security (1: light client, 2: wallet default, 3: exchange level)
var key = function(seed, index, length) {
...
    return key;    // private key
}
 ```
> 1. have your Private Key ready.
> 2. divides the Private Key into 'L' segments, where `L = security * 27`.
> 3. Hash all segments as a whole. The product is called `digest`.
> 4. Hash `digest` twice. The product is called `address`.

![address_gen1.png](https://qiita-image-store.s3.amazonaws.com/0/187795/b6e44924-6bc9-71ab-5360-d8db7069879a.png)

# Signature
Signature is used to sign anything(=signed data usually bundle) on tangle that belongs to you with your private key.

> 1. have your Private Key ready.
> 2. divides the Private Key into 'L' segments, where `L = security * 27`.
> 3. For each *i*-th segment, hash N_*i* times, where N_*i* is caluculated as followed:
> > **How to get N**
> > For each *i*-th tryte of the **Signed Data**, get decimal 'd' of the tryte. Converter [here](https://qiita-image-store.s3.amazonaws.com/0/187795/e325fe61-7773-8e64-46e0-2e98d66aacf4.png). e.g) tryte[9] corresponds to `d=0`, [A] is to `d=1`...[M] is to `d=13`, L is to `d=-13`...Y is to `d=-2`, Z is to `d=-1`).
> > Formula: *N_i = 13 - d*
> 4. Those hashed segments are called **Signature** together.

![signing1.png](https://qiita-image-store.s3.amazonaws.com/0/187795/303a022c-19ef-fdd9-4125-700f51a8a005.png)

# Validation (address Re-generation )
> 1. have your Signature ready.
> 2. divides the Signature into 'L' segments, where `L = security * 27`.
> 3. For each *i*-th segment, hash M_*i* times, where M_*i* is caluculated as followed:
> > **How to get M** (basically main idea is same as N.)
> > For each *i*-th tryte of the **Signed Data**, get decimal 'd' of the tryte. Converter [here](https://qiita-image-store.s3.amazonaws.com/0/187795/e325fe61-7773-8e64-46e0-2e98d66aacf4.png). e.g) tryte[9] corresponds to `d=0`, [A] is to `d=1`...[M] is to `d=13`, L is to `d=-13`...Y is to `d=-2`, Z is to `d=-1`).
> > Formula: *M_i = 13 + d*
> 4. Hash those segment together and get `digest`.
> 5. Hash the `digest` twice.
> 6. Check if the product of step 5 matches the address of the signed data(usually Bundle).

![validate1.png](https://qiita-image-store.s3.amazonaws.com/0/187795/e4f2e7de-2a46-e25e-4fe9-25a024ad7eba.png)

# Signed Data
　Signature is used to sign your input address when you spend. And signature is stored in the bundle that spends the signed input. Signature data (length = security * 2187 tryte) is stored at `signatureFragment`. (Note that `signatureFragment`'s capacity is 2187 tryte, so the larger security, the more transactions for storing signature are necessary to be included in the bundle.)
 Signed data mentioned above refers to the bundle hash (81 tryte) that includes the signature.
 （Strictly speaking, signed data is called normalized bundle hash, which is slightly incremented bundle hash.）

 ![singed_data.png](https://qiita-image-store.s3.amazonaws.com/0/187795/f0c3dbb7-503c-51a7-64c3-579162c6fb71.png)

　*Signed Data（署名されるデータ）*は上図のように、data[0]、data[1]、data[2]というNormalized Bundleを分割したものを使う。27セグメントを何回ハッシュに通すかはSigned Dataの27トライトに対応していた。この27トライトが上図のdata[i]である。`security`との関係だが、`security`レベルによってPrivate keyの長さ（`= security * 2187`）は変化し、それによってセグメントの数（`=security*27`）が変化したことを思い出してほしい。`security=2`の場合、セグメント数は54個、つまりdata[0]とdata[1]合わせた54トライトと対応させてセグメントを一つずつN回ハッシュにかける。（Nの求め方は上で説明した。）
　Bundle生成の話と繋げる。securityによって署名の数が増えた理由は、署名①はdata[0]、署名②はdata[1]、という風に署名されるデータの分担を行なっていたからである。
　securityが4以上の場合はNormalized Bundleが81トライトという理由で、data[2]の次はdata[0]という風にループさせて署名されるデータを作る。

## Normalized Bundle
　口では説明し難いのでソースコードにコメントをふった。マニアックすぎるので正直言って無視して良い。
　なるべくトライト表の中間にあるアルファベット(..IJKLMNOP..）を[ABCやXYZ]方面へ寄せバランスを取らせている。しかし、その意義は残念ながら分からない。

```java:Bundle.java
/**
     * Normalized the bundle.
     * return the bundle each tryte is written in integer[-13~13]
     *
     * @param bundleHash BundleのHash。
     * @return normalizedBundle A normalized bundle hash.
     */
    public int[] normalizedBundle(String bundleHash) {

        //  normalized Bundle 81トライト。
        //  これに値を当てはめてreturnする。
        int[] normalizedBundle = new int[81];

        //  Bundle Hash(81 トライト)を３つのセクションに分割(1つあたり27トライト)
        for (int i = 0; i < 3; i++) {

            // sum セクションの合計。
            long sum = 0;

            //  1セクション内を１トライトずつ確認し、
            //  そのトライトに対応する整数[-13~13]を求めてセクションの合計に加えていく。
            for (int j = 0; j < 27; j++) {

                //  sum += value
                //  value = normalizedBundleのi*27+j番目のトライト[9~Z]を数字[-13~13]に変換
                sum += 
                    //  トライトに対応する整数[-13~13]をnormalizedBundleの該当ポジションに代入。
                    (normalizedBundle[i * 27 + j] = 

                        //  トライト[9ABC...Z]を整数[-13~13]に変換
                        Converter.value(Converter.tritsString("" + bundleHash.charAt(i * 27 + j)))
                    );
            }

            // もし、sumが0以上なら、
            if (sum >= 0) {

                //  sumが0になるまで
                while (sum-- > 0) {

                    //  セクション内で[-13]以上のトライトを-1する。一つ-1したら最初から。
                    for (int j = 0; j < 27; j++) {
                        if (normalizedBundle[i * 27 + j] > -13) {
                            normalizedBundle[i * 27 + j]--;
                            break;
                        }
                    }
                }

            //  もし、sumが0以下なら、
            } else {

                //  sumが0になるまで
                while (sum++ < 0) {

                    //  セクション内で[13]以下のトライトを+1する。一つ+1したら最初から。
                    for (int j = 0; j < 27; j++) {

                        if (normalizedBundle[i * 27 + j] < 13) {
                            normalizedBundle[i * 27 + j]++;
                            break;
                        }
                    }
                }
            }
        }

        return normalizedBundle;
    }
```

# Risk of Address Reuse
　先日、IOTA Supportからウォレットの"タングルにアタッチ"機能とセキュリティの関係について[記事](http://iotasupport.com/how-addresses-are-used-in-IOTA.shtml)が出され、個人的にいくつかの疑問点が払拭されたので説明する。まず、署名方法の中のNの求め方を見てほしい。

>**Nの求め方**
**署名されるデータ（Signed Data）**のi番目のTryteを見る（A〜Zか9、の全27種類）そのTryteが[この表](https://qiita-image-store.s3.amazonaws.com/0/187795/e325fe61-7773-8e64-46e0-2e98d66aacf4.png)のdecimalのどの数字dに対応しているか確認する。（9ならd=0、Aならd=1...Mならd=13、Lならd=-13...Yならd=-2、Zならd=-1）
*Ni = 13 - d*
i番目のセグメントをNi回ハッシュ関数に通す。

　データ内i番目のトライトがによって、Prvate Key内のi番目のセグメントにハッシュ関数が少なくなる場合がある。例えば**Mなら0回**だ（13-13=0）。もし、署名されるデータがMを多く含むものであった場合、電子署名はPrivate Keyの部分的な生データを総当たりで求められてしまう。つまり、同じアドレスを再利用して様々なデータにそのアドレス名義で署名を行えば行うほど、生のPrivate Keyがハックされてしまう可能性が高まる。

これを防ぐためにすることは、**一度使ったアドレスは二度と使わない。**
言い換えると、**一度支払いに使ったアドレスに入金しない。**
厳密に言うと、**一度入力アドレスとして使ったアドレスはもう一度入力アドレスとして使ってはならない。**
もっと厳密に言うと、**一度署名に使ったPrivate Keyを別の署名に使わない。**
　技術的なことが分からなくても、とりあえずIOTAで送金をするたびに、ウォレットの"受取"のところからアドレスを生成するだけで防げる。

## 量子計算耐性
　このような署名メカニズムを採択した背景には量子コンピュータ耐性を持たせると言う目的がある。[Winternitz one-time signature](https://eprint.iacr.org/2011/191.pdf)という論文でこの署名方法について知ることができるらしい。うーん。分からん。[量子コンピュータ耐性があるランポート署名について解説&シェルスクリプトで実装してみた](https://qiita.com/onokatio/items/689965fa484d40d851ce)という記事も参考になるかもしれない。


# Trits/Tryte to Decimal Table
<img width="184" alt="tryte.png" src="https://qiita-image-store.s3.amazonaws.com/0/187795/e325fe61-7773-8e64-46e0-2e98d66aacf4.png">


#参考文献
IOTA公式ライブラリ [https://github.com/iotaledger](https://github.com/iotaledger)
本記事はSigningという部分。JavaScriptなら[signing.js](https://github.com/iotaledger/iota.lib.js/blob/master/lib/crypto/signing/signing.js)、Javaなら[Signing.java](https://github.com/iotaledger/iota.lib.java/blob/master/jota/src/main/java/jota/utils/Signing.java)から。


