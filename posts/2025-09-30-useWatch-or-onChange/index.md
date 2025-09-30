# [tochisuke docs](https://tochisuke221.github.io/)

## useWatch vs onChange

## 背景
フォーム開発でよくあるケース：
「入力Aが変わったら入力Bを更新する」「選択肢に応じて別フィールドの候補を絞り込む」。

この要件に対しては2つのアプローチが考えられる
- React Hook Form ではこれをuseWatch で依存関係を監視する
- onChange関数で副作用的に依存フィールドの値を変化させる

それぞれの性質とトレードオフについて考えみる


## useWatch で依存フィールドを監視する

**特徴**
- 宣言的に「この値を監視する」と書ける。（後述）
- 複数フィールドに依存するロジックをひとまとめにできる。
- 再レンダリング範囲を限定できる（フォーム全体ではなく監視対象だけ）。

**実装例**

```ts
const country = useWatch({ control, name: "country" });
const { setValue } = useFormContext();

useEffect(() => {
  if (country === "JP") {
    setValue("city", "Tokyo");
  }
}, [country, setValue]);
```

## onChange関数で副作用的に依存フィールドの値を変化させる
**特徴**
- 命令的に「この入力が変わった瞬間に処理する」と直感的に書ける。
- 単一フィールド依存のシンプルなケースでは簡潔でわかりやすい。

**実装例**

```ts

const handleChangeSelect = (e) => {
  const country = e.target.value;
  setValue("country", country);
  if (country === "JP") {
    setValue("city", "Tokyo");
  }
};

<select
  {...register("country")}
  onChange={handleChangeSelect}
>
  {/* ...options */}
</select>


```

## トレードオフの表
| 観点      | useWatch      | onChange        |
| ------- | ------------- | --------------- |
| 記述スタイル  | 宣言的（依存関係を監視）  | 命令的（イベント単位で処理）  |
| ロジックの配置 | useEffect に集約 | フィールドごとのハンドラに分散 |
| 再レンダリング | 監視対象のみ        | イベント発生ごと        |
| 可読性     | 複雑な依存を整理しやすい  | 単純な処理は直感的で読みやすい |
| 保守性     | ロジック集中で変更に強い  | 分散しやすくスケールしにくい  |
| 適用規模    | 中〜大規模フォーム     | 小規模フォーム         |


## 補足1: 宣言的って何？

React の世界観でよく言う「宣言的」は、UI が「状態の結果としてどうなるか」を記述する ということ。

```ts
{isOpen && <Dialog />}
```

→ 「isOpen が true なら Dialog を表示する」という宣言。

DOM を直接操作して document.createElement するのは命令的（imperative）。

つまり、、、

```ts
<select onChange={handleChangeSelect}>...</select>
```

「このイベントが起きたら関数を呼ぶ」という ルール を宣言しているので、DOM を imperatively 操作するのに比べれば宣言的だがhandleChangeSelect の中では
「こうしろ」と命令的な処理を実行している。

つまり 書き方は宣言的っぽいが、中身は命令的である。

useWatchは、「このフィールドを監視する」という依存関係を明示していて、状態変化に基づく振る舞いを宣言的に記述している。
「状態の関係性をコードでモデル化する」点で onChange よりは宣言的である。

```ts
const country = useWatch({ control, name: "country" });

useEffect(() => {
  if (country === "JP") {
    setValue("city", "Tokyo");
  }
}, [country]);
```

## 補足2: useWatchで監視対象が増えた場合の対応

useWatchの多用で監視対象が増えてしまった場合、**著しく可読性が下がる**。
こうした場合はどのようにすべきか。


1. 複数フィールドをまとめて監視する

```ts
const {
    email,
    phoneNumber,
    adminFirstName,
    adminLastName,
  } = useWatch<FormValues>({
    name: [
      'email',
      'phoneNumber',
      'adminFirstName',
      'adminLastName',
    ],
  })
```

2. カスタムフックにまとめる
- 監視と変化ロジックをまとめてしまう？

``` ts
export const useSyncOffice = () => {
  const { control, setValue } = useFormContext<FormValues>()

  // companyId と officeId をまとめて監視
  const { companyId, officeId } = useWatch({
    control,
    name: ['companyId', 'officeId'],
  })

  useEffect(() => {
    // companyId が変わったら officeId をリセット
    if (officeId && companyId === null) {
      setValue('officeId', null)
    }
  }, [companyId, officeId, setValue])
}
```

これで綺麗にならないくらい監視が多い場合は、そもそも使うユーザもそのくらい複雑な副作用は分かりにくくなる恐れがあるので
デザイン修正や要件見直しからやることも検討すると良いかも。

いずれにせよonChangeとuseWatchの利用はメリデリがあるので、何を解決したいかベースで考えると良さそう
