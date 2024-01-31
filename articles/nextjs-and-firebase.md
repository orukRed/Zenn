---
title: "Next.jsとfirebaseを連携させて画像投稿アプリを作る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Next.js", "firebase", "react"]
published: true
---

## はじめに

この記事ではNext.jsとfirebaseを連携させて画像投稿アプリを作ります。
どちらかというと、躓いた点のtips集という側面が強いです。
ちなみに後述しますが、ほぼクライアントコンポーネントで作っています。

## この記事の対象者

- 環境構築は自分でできる人
- Next.js/firebaseを学び始めて習作作りたい人
- Next.jsでWebアプリ作成～firebaseのホスティングまでで何かしら躓いた点がある人


## 使用技術

- Next.js 14.0.4（app router使用。）
- @nextui-org/react: ^2.2.9
- fontawesome: ^5.6.3
- Windows11

## 作るもの

ログインして画像を投稿・表示するだけの簡単なものです。
これでも割と躓いたので……
![画像投稿アプリ](/images/articles/nextjs-and-firebase/app.png)

## 設計方針

Next14.0.4なので以下のものが使えますが、今回使うのはApp routerのみです。


- [App Router](https://Next.js.org/docs/app)
- [サーバーサイドコンポーネント](https://Next.js.org/docs/app/building-your-application/rendering/server-components)
- [Server Actions](https://Next.js.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)
- [Route Handlers](https://Next.js.org/docs/app/building-your-application/routing/route-handlers)

等いろいろあるのですが
理由は以下の通りです。

### App router

Next.js13.4からApp Routerが使えるようになりました。
今までは`pages`ディレクトリにフォルダを作成すると、そのファイル名がURLになっていましたが、
App Routerでは`app`ディレクトリにフォルダを作成します。
![ディレクトリ](/images/articles/nextjs-and-firebase/approuter.png)
`pages`でのルーティングと違いがほぼない（ように感じた）ので、今回はApp Routerを使います。

### サーバーサイドコンポーネント、 Server Actions、 Route Handlersを使わない理由

**そもそもfirebase SDKはクライアント用です。**
**サーバーサイド側で使いたいなら、Firebase Admin SDKを使う必要があります。**
**例として、firebaseのAuthenticationを使いログイン処理を無理やりクライアントコンポーネントで書いた場合、サーバーサイドコンポーネントでは認証されてないものとして扱われます。**

また、今回くらいの軽量アプリならサーバーサイドの機能を使う必要はないと感じました。
セキュリティの担保ならfirebaseのルールで対応できます。
<!-- ちなみにRoute Handlers使うくらいならOpenAPIで定義した方がいいのかも……？（要調査） -->

ちなみにgetServerSidePropsはapp routerでは使えない？みたいなので注意。
ネットやgithub copilotの情報はapp routerとpages両方の情報が混ざってます。

## firebaseの設定

firebaseのログインしてプロジェクト作成後、SDKの取得をしたらクライアント側で使えるようにします。

初めに、firebase SDKは使用前に初期化する必要があります。
基本的にfirebase SDK呼び出す前に、以下のファイルからinitFirebase()を呼べばOKです。
とはいえアプリのLoginページで一度だけ呼べばOKそうです。
それにもっといい書き方あるかも🤔

```javascript:client.js
const firebaseConfig = {
  apiKey: "",
  authDomain: "",
  projectId: "",
  storageBucket: "",
  messagingSenderId: "",
  appId: "",
  measurementId: ""
};
// Initialize Firebase
export let firebaseApp = !getApps().length ? initializeApp(firebaseConfig) : getApps()[0];
export function initFirebase() {
  firebaseApp = !getApps().length ? initializeApp(firebaseConfig) : getApps()[0];
}
// const analytics = getAnalytics(firebaseApp);
```

任意のタイミングで使えるように/components/firebase.js等に置いておくとよいです。

ちなみにGithub Actionsとかでデプロイする場合は、API KEYを環境変数に設定しておく必要があります。

## ファイル構造

Next.jsのファイル構造は以下のようになっています。
![alt](/images/articles/nextjs-and-firebase/explo.png)

### app

ルーティングするファイル。
`フォルダ名＝パス`となります。
また、ファイル名は`page.tsx`で固定です。
また、各コンポーネントの頭文字は大文字にする必要があります。
```typescript
//page.tsx
export default function Hoge() {
  return <div>Page</div>;
}

//他ファイルやコンポーネントから以下のように呼び出せる。
export default function Fuga() {
  return <Hoge />;
}
```

### components

ルーティングしたくないtsxファイル（モーダル等のコンポーネント類）を作った場合はこちらに。
また、`.ts`のような汎用関数として定義しただけのもこちらに。

## ログインページ

ソース全文は以下。（後で重要な部分は解説するので見なくてOK）

```typescript
export default function LoginForm({}: {}) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [isAuthing, setIsAuthing] = useState(false); //ログイン処理中のフラグ
  const [now, formattedDate] = getDate();

  //ログインパスワードとemailが一致してログイン処理中のフラグ
  initFirebase();
  const router = useRouter();
  const authenticationUser = async (email: string, password: string) => {
    const auth = fireAuth.getAuth(); //初期化処理
    fireAuth.setPersistence(auth, fireAuth.browserSessionPersistence);
    const db = firestore.getFirestore(firebaseApp);

    setIsAuthing(true);
    console.log('email:' + email);
    console.log('password:' + password);

    fireAuth
      .signInWithEmailAndPassword(auth, email, password)
      .then(async (userCredential) => {
        const user = userCredential.user;
        await auth.updateCurrentUser(userCredential.user);
        await firestore.setDoc(firestore.doc(db, 'log_login', `${formattedDate}_login success`), { message: 'logged in', email: email, password: password, ipAddress: await getIpAddress() });
        console.log(user);
        router.push('/view');
      })
      .catch(async (error) => {
        await firestore.setDoc(firestore.doc(db, 'log_login', `${formattedDate}_login fail`), { errorStack: error.stack, errorMessage: error.message, email: email, password: password, ipAddress: await getIpAddress() });
        console.log(await error);
        setIsAuthing(false);
        alert('ログインに失敗しました。');
      });
  };
  initFirebase();

  return (
    <div className=' flex items-center justify-center  py-12 px-4 sm:px-6 lg:px-8'>
      <div className='max-w-md w-full space-y-8'>
        <input type='hidden' name='id' value='userId' />
        <Input name='email' className='mt-1 block w-full py-2 px-3   rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm' type='email' label='Email' onChange={(e) => setEmail(e.target.value)} />
        <Input name='password' className='mt-1 block w-full py-2 px-3    rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm' placeholder='Password' type='password' onChange={(e) => setPassword(e.target.value)} />
        <Button
          className='group relative w-full flex justify-center py-2 px-4 border border-transparent text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500'
          onClick={async () => {
            await authenticationUser(email, password);
          }}
          type='submit'
        >
          Login
        </Button>
      </div>
      <ModalSpinner isLoading={isAuthing} />
    </div>
  );
}

```

UI部分については割愛します。
任意のCSS/UIフレームワークで好きなように作ってください。

### Authenticationの設定

事前にfirebaseから認証方法の設定が必要です。
色々あるのですが今回はメールアドレスとパスワードを選択。
今回はユーザー作成まではやらないので、任意のメアドとパスワードを事前に登録しておきましょう。

### ログイン処理

`authenticationUser`でfirebaseと接続してログイン処理を行います。

```typescript
  initFirebase();
  const router = useRouter();
  const authenticationUser = async (email: string, password: string) => {
    const auth = fireAuth.getAuth(); //初期化処理
    fireAuth
      .signInWithEmailAndPassword(auth, email, password)
      .then(async (userCredential) => {
        router.push('/view');
      })
      .catch(async (error) => {
        console.log(await error);
      });
  };
```

`signInWithEmailAndPassword`関数でメアドとパスワードによる認証ができます。
~~今回長々と処理書いてますが~~ 実際には`signInWithEmailAndPassword`と、成功したらリダイレクトするくらいでOKです。
強いて言うなら、万が一のためにログイン失敗時にエラーをログに残すくらいでしょうか。（パスワードを書き込む場合は暗号化できるとより良い）
成功した場合は`userCredential.user`にユーザー情報が入りますので、firestoreにログイン履歴など残す場合はここで使えます。
（今回はメアドとパスワードしか保存していませんが）
ちなみにapp routerでは`'next/navigation'`を使ってください。`'next/router'`は使えません。

## 画像投稿ページ

こんな感じの画面作ります。

![alt](/images/articles/nextjs-and-firebase/throw.png)


### 画像クリックでダイアログ出す

このページは画像クリックで画像選択用のダイアログが開くようになっています。
※わかりやすさ優先して一部コード省略しています。全コードはGitHubを見てください。

```typescript
const [previewSrc, setPreviewSrc] = useState<string | null>(null);

function SelectImage({ previewSrc, setPreviewSrc }: ChildComponentProps) {
  const fileInputRef: any = useRef(undefined);
  const callFileSelector = () => {
    if (fileInputRef && fileInputRef.current) {
      fileInputRef.current.click();
    }
  };
  const selectFile = (e: any) => {
    const file = e.target.files[0];
    const reader = new FileReader();
    reader.onloadend = () => {
      if (typeof reader.result === 'string') {
        setPreviewSrc(reader.result);
      }
    };
    if (file) {
      reader.readAsDataURL(file);
    }
  };
  selectedImage = previewSrc;
  return (
    <>
      <div className={styles.checkerboard} onClick={callFileSelector}>
        <input type='file' accept='.png, .jpg, .jpeg, .gif .webp' style={{ display: 'none' }} ref={fileInputRef} onChange={selectFile} />
        <div className='text-xl text-white  shadow-lg font-bold'>ファイルを選択してください</div>
        <div className='flex justify-center items-center'>
          <FontAwesomeIcon icon={faImage} size='10x' />
        </div>
      </div>
    </>
  );
}
```

流れとしては以下です。
1. inputにref属性、onChangeでselectFile（ファイル選択ダイアログから画像を選んだ時のイベント）を追加する
2. 任意の場所（今回はcallFileSelector関数）から`fileInputRef.current.click()`でinputをクリックした処理を呼び出し、ファイル選択ダイアログを呼び出す
3. 任意の画像を選択して、selectFile関数が呼ばれる
4. `e.target.files[0]`に選択した画像が入るので、FileReaderクラスのreadAsDataURL関数で画像を読み込む
5. onloadendで、読み完了後にsetPreviewSrcでreader.resultに格納されているbase64形式の画像を格納
6. `<img src={previewSrc}`で格納した画像を読み込む


### 透明色のCSS定義

上記画面の黒とグレーのタイル背景は（ペイントソフトの透過色として使われるようなもの）はCSSで定義しています。
おそらくプリセットのようなものはないので自前で定義しましょう。

```css
.checkerboard {
  background-image: linear-gradient(45deg, rgba(128, 128, 128, 0.5) 25%, transparent 25%),
    linear-gradient(-45deg, rgba(128, 128, 128, 0.5) 25%, transparent 25%),
    linear-gradient(45deg, transparent 75%, rgba(128, 128, 128, 0.5) 75%),
    linear-gradient(-45deg, transparent 75%, rgba(128, 128, 128, 0.5) 75%);
  background-size: 20px 20px;
  background-position: 0 0, 0 10px, 10px -10px, -10px 0px;
}
```

ちなみにAIに聞けばサクッと答えてくれました。
chatGPTなりGithub Copilotなり入れておくとこういう時便利そう。

### firestore,storageへの登録処理

コードは以下です。
今回も簡略化して一部のみ抜粋しています。

```typescript
let db = firestore.getFirestore(firebaseApp);

//storageへの画像の追加を行う
const storagePath = `images/${image.userId}_${formattedDate}`; //storageのパス
const s = storage.getStorage(firebaseApp);
const storageRef = storage.ref(s, storagePath);
const uploadTask = storage.uploadString(storageRef, previewSrc, 'data_url');

//imageのデータを作成
image.userId = -1; //仮値
image.title = name;
image.description = description;
image.filePath = (await uploadTask).ref.fullPath;
image.ipAddress = await getIpAddress();
image.createdAt = now;
image.updatedAt = null;
image.deletedAt = null;

//データの追加
await firestore.setDoc(firestore.doc(db, 'images', `${image.userId}_${formattedDate}`), image);
await firestore.setDoc(firestore.doc(db, 'log_image', formattedDate), { message: 'registerImage' });
```

画像登録なので、storage（画像）とfirestore（NoSQL）に登録します。
流れとしては
firestoreから画像のメタデータ取得→メタデータ内の画像へのパスをもとにstorage取得
という感じです。
画像の取得だけならstorageのみでOKですが、将来的にはユーザー管理などもできるとよいのでこのような設計にしています。

### ルールの設定

firestore,storageはルール（セキュリティのようなもの）を設定しないと使えません。
以下のように、セキュリティを設定してあげてください。
基本的には、read,create,delete,update等があるのでコレクションの用途に合わせて一つずつ許可をしていく感じです。
request.authで現在認証中かどうかの取得ができます。

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    function isAuthenticated() {
      return request.auth != null;
  	}
    // imagesコレクションは認証済みのユーザーのみ読み書き可能
    match /images/{document=**} {
      allow read, create: if isAuthenticated();
      allow update : if false;
    }
  }
}
```

## 画像取得ページ作成

下記のような処理でfirestoreやstorageから画像を取得する処理を書けばOKです。
私はfirestoreにstorageへのパスを入れているのでそれを使ってstoregeから画像を取得しています。

```typescript
  const fetchDataFromFirestore = () => {
    const db = firestore.getFirestore(firebaseApp);
    const imageCollection = firestore.collection(db, 'images');
    const query = firestore.query(imageCollection, firestore.where('isPrivate', '==', false), firestore.orderBy('createdAt', 'desc'), firestore.limit(limit));

    return firestore.getDocs(query).then((imageSnapshot) => {
      const imageList = imageSnapshot.docs.map((doc) => doc.data());
      return imageList;
    });
  };
  const fetchDataCountFromFirestore = () => {
    const db = firestore.getFirestore(firebaseApp);
    const imageCollection = firestore.collection(db, 'images');
    const query = firestore.query(imageCollection, firestore.where('isPrivate', '==', false), firestore.orderBy('createdAt', 'desc'));
    return firestore.getCountFromServer(query).then((count) => {
      return count.data().count;
    });
  };
  const fetchImagesFromStorage = (imageList: firestore.DocumentData[]): Promise<firestore.DocumentData[]> => {
    const returnList = imageList.slice();
    const promises = returnList.map((image) => {
      const storageRef = storage.ref(storage.getStorage(), image.filePath);
      return storage.getDownloadURL(storageRef).then((url) => {
        image.url = url;
      });
    });
    return Promise.all(promises).then(() => returnList);
  };
```

### onAuthStateChangedで認証完了まで待機する

この時、onAuthStateChangedで認証完了するまで待機しないと、認証されていない状態でfirestoreやstorageにアクセスしてしまいエラーになります。
（ルールでrequest.authで認証中かどうかを判定している場合）

onAuthStateChangedは以下のように、useEffectを使い呼び出します。
onAuthStateChangedは認証済みの場合userを返却しますので、userが存在する場合のみfirestoreやstorageからデータを取得するようにしています。
最後にunsubscribeで、複数回呼ばれないようにしてください。

```typescript
  useEffect(() => {
    const unsubscribe = fireAuth.onAuthStateChanged(auth, (user: any) => {
      if (user) {
        fetchDataFromFirestore().then((imageList) => {
          fetchImagesFromStorage(imageList).then((updatedImageList) => {
            setImageList(updatedImageList);
          });
        });
        fetchDataCountFromFirestore().then((count) => {
          setImageCount(count);
        });
      }
    });
    return () => unsubscribe();
  }, []);
```

### useStateとuseEffectでの無限ループ

**下記コードの場合、無限ループが発生します**
**useEffectの第二引数に変数を入れた場合、変数が更新されるたびにuseEffectが呼ばれます。**
**つまりuseEffect内でimageListを更新して、それをトリガーにuseEffectが...となります。**
fierbase優良プランなどの場合大事故になるので気を付けましょう。

```typescript
  useEffect(() => {
    const unsubscribe = fireAuth.onAuthStateChanged(auth, (user: any) => {
      if (user) {
        fetchDataFromFirestore().then((imageList) => {
          fetchImagesFromStorage(imageList).then((updatedImageList) => {
            setImageList(updatedImageList);
          });
        });
        fetchDataCountFromFirestore().then((count) => {
          setImageCount(count);
        });
      }
    });
    return () => unsubscribe();
  }, [imageList]);
```

### ページネーション処理

画像が増えてくるとページを区切る必要があります。
UIフレームワークにページネーション用のコンポーネントがあると思うので、それを使うとよいです。

- 1ページに表示する画像数
- 現在のページ数（URLパラメータで渡す）
- 画像の総数
があれば実装できます。
ページネーションの
初期値に現在のページ
総ページに(画像の総数/1ページに表示する画像数)を整数型で切り上げればOKです。

```typescript
export default function PaginationPage({ imageCount, imageLimit, currentPage }: { imageCount: number; imageLimit: number; currentPage: number }) {
  const searchParams = useSearchParams();
  const router = useRouter();
  const handlePageChange = (selectedPage: number) => {
    router.push(`/view?page=${selectedPage}`);
  };

  return (
    <>
      <Pagination
        showControls={true}
        total={Math.ceil(imageCount / imageLimit)}
        initialPage={currentPage}
        onChange={(value) => {
          handlePageChange(value);
        }}
      />
    </>
  );
```

## 完成したらfirebaseの設定～デプロイまで

ここら辺は以下を実行して手順に従ってください。

```shell
login firebase
init firebase
```

## ホスティングについて

Next.jsのホスティングの場合、なぜか通常のホスティングではうまくいかず、firebaseデフォルトの画面が表示されるのみでした。
以下の手順試すといけました。

<https://firebase.google.com/docs/hosting/Next.js?hl=ja>

```shell
firebase experiments:enable webframeworks
firebase init hosting
firebase deploy
```

## GithubActionsでの自動デプロイ

エラー出ちゃったので注意。
私はclient.jsにAPI KEY設定してgithubにで.gitignoreでプッシュしないようにしていたのですが、当然GithubActionsでのデプロイ時にはAPI KEYがないのでエラーになりました。
環境変数を設定しておく必要があります。

## 今後のTODO

まだ未実装の機能がいくらかあります。
今後も追加予定なので、追加するたびに更新します。

- [ ] ユーザーごとのマイページ追加
- [ ] 削除機能の追加（論理削除）
- [ ] Github actions通るようにしておく（環境変数追加）
- [ ] エミュレータを使いローカルでの動作検証をもっと自由にする
- [ ] ログイン方法をメアドとパスワード以外から増やしたり二段階認証を追加する
- [ ] テストコード作る

## 今回作ったページのソースコード

今回作ったアプリのソースコードです。
プルリクやご指摘いただけると幸いです！
https://github.com/orukRed/image-thrower/tree/zenn

## 参考URL

https://qiita.com/KosukeSaigusa/items/18217958c581eac9b245
https://qiita.com/UfUzR64rRzYa_Question/questions/7ccaa5d78b1de66d2e50
https://zenn.dev/hiroga/articles/9aa555422a00cf