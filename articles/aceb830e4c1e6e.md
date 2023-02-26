---
title: "cognitoの各種認証画面を独自実装する"
emoji: "🔱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cognito", "nextjs", "mui"]
published: false
---

# はじめに

Cognitoの認証を利用するときに、HostedUIとかはいいけど、やっぱり自分で作りたいと思う時があります。
HostedUIはサービスのイメージに合わなかったりするためです。
また、AmplifyUIもありますが、これも利用しなくても画面は実装していきます。

そこで、今回はUIフレームワークの1つであるMUIを使ってサインイン画面やパスワード変更画面などを実装していきます。
画面は`MUI`をベースに作られていますが、認証まわりの処理は単一のファイルとして切り出されているため、`React`や`Next.js`を利用する場合なら、
ほぼほぼコピペで利用できるかと思います。

主に参考にしたのは以下の記事になっています。

https://mseeeen.msen.jp/react-auth-with-ready-made-cognito/

この記事の中でも述べられていますが、`aws-amplify`の中の`Auth`モジュールを利用して
認証まわりの処理を組んでいきます。
このモジュールは`Amplify`とは何も関係がなく単純に認証モジュールとして便利なため利用します。



ログインの見た目はMUIを使っているので、以下のページのを参考にした。

https://qiita.com/shunnami/items/98a341d2ac20775241ad

# 利用時のイメージ・この記事の対象読者

利用時のイメージは次のようになっています。
最初の画面はサービスのトップページを想定しています。

![](https://storage.googleapis.com/zenn-user-upload/c0110fa4ebec-20230225.png =600x)
*サインイン前の状態*

画面右上のアカウントボタンを押下すると、サインイン画面に遷移します。
ユーザーの情報を入力して認証が成功すると、トップページに遷移します。

![](https://storage.googleapis.com/zenn-user-upload/2fd3e250ffbb-20230225.png =600x)
*サインイン画面*

認証済みの場合は、認証後の画面を用意する想定です。
（今回は認証済みの場合「サインイン後」という文字になるようにしてあります。）

![](https://storage.googleapis.com/zenn-user-upload/51b333dfab41-20230225.png =600x)
*サインイン後の状態*

## この記事の対象読者

- `React`か`Next.js`を利用する方
- `cognito`の認証を利用する方
- `Amplify`を利用せずに、`CloudFront`やその他の方法で独自にサイトをホスティングする方


## デプロイ環境

利用したフロントエンドの主なライブラリ

- react: 18.2.0
- Next.js: 13.2.1
- MUI: 5.10.12
- react-hook-form: 7.43.2
- aws-amplify: 5.0.15

# コード

今回のコードは[このリポジトリ](https://github.com/gsy0911/zenn-cognito-frontend-auth)に載せてありますので適宜参考にしてください。

以下から簡単に内容を紹介していきます。

## ディレクトリ構成

```text
frontend/
├── components
│  ├── model
│  │  └── Header
│  │     ├── Header.tsx 👈 見た目を少しよくするために用意したヘッダー
│  │     └── index.ts
│  └── page
│     ├── Auth
│     │  ├── AuthBasePage.tsx 👈 a
│     │  ├── PasswordChange.form.tsx 👈 a
│     │  ├── PasswordForget.form.tsx 👈 a
│     │  ├── PasswordReset.form.tsx 👈 a
│     │  ├── SignIn.form.tsx 👈 a
│     │  └── SignInComplete.form.tsx 👈 a
│     └── index.ts
├── lib
│  └── cognito-auth.tsx 👈 a
├── next.config.js
├── package-lock.json
├── package.json
├── pages
│  ├── _app.tsx
│  ├── _document.tsx
│  ├── index.tsx
│  ├── password-change
│  │  └── index.tsx 👈 「パスワード変更」を表示
│  ├── password-forget
│  │  └── index.tsx 👈 「パスワード忘れ画面」と「パスワード再設定」を表示
│  └── signin
│     └── index.tsx 👈 「サインイン画面」と「初回サインイン完了画面」を表示
└── tsconfig.json
```

[//]: # (今回は 👈 でマークのあるファイルを変更・追加します。)

## cognitoの設定

## 認証用フックの準備

今回の大事なコードの1つである`cognito-auth.tsx`を記述します。
認証の各種関数や状態を管理するようになっています。
このファイルは、[参考記事](https://mseeeen.msen.jp/react-auth-with-ready-made-cognito/)内容にいくつかの関数を追加しています。

ファイルは長いので、見たい場合はアコーディオンを開いてください。

::::details 認証用フックの詳細

```typescript jsx: cognito-auth.tsx
import {Auth, Amplify} from 'aws-amplify';
import React, {createContext, useContext, useEffect, useState} from 'react';

const AwsConfigAuth = {
  region: process.env.NEXT_PUBLIC_COGNITO_REGION,
  userPoolId: process.env.NEXT_PUBLIC_COGNITO_USER_POOL_ID,
  userPoolWebClientId: process.env.NEXT_PUBLIC_COGNITO_WEB_CLIENT_ID,
  authenticationFlowType: 'USER_SRP_AUTH',
}

Amplify.configure({Auth: AwsConfigAuth});

interface UseAuth {
  isLoading: boolean;
  isAuthenticated: boolean;
  username: string;
  currentAuthenticatedUser: () => Promise<any>;
  signUp: (username: string, password: string) => Promise<Result>;
  confirmSignUp: (verificationCode: string) => Promise<Result>;
  signIn: (username: string, password: string) => Promise<Result>;
  signInComplete: (username: string, oldPassword: string, newPassword: string) => Promise<Result>;
  signOut: () => void;
  changePassword: (user: any, oldPassword: string, newPassword: string, newPasswordConfirm: string) => Promise<Result>;
  forgetPassword: (email: string) => Promise<Result>;
  resetPassword: (username: string, code: string, newPassword: string, newPasswordConfirm: string) => Promise<Result>;
}

interface Result {
  success: boolean
  message: string
  hasChallenge?: boolean
  challengeName?: string
}

const authContext = createContext({} as UseAuth);

export const ProvideAuth: React.FC<{ children: React.ReactNode }> = ({children}) => {
  const auth = useProvideAuth();
  return <authContext.Provider value={auth}> {children} </authContext.Provider>;
};

export const useAuth = () => {
  return useContext(authContext);
};

const useProvideAuth = (): UseAuth => {
  const [isLoading, setIsLoading] = useState(true);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');

  useEffect(() => {
    Auth.currentAuthenticatedUser()
      .then((result) => {
        setUsername(result.username);
        setIsAuthenticated(true);
        setIsLoading(false);
      })
      .catch(() => {
        setUsername('');
        setIsAuthenticated(false);
        setIsLoading(false);
      });
  }, []);

  const signUp = async (username: string, password: string) => {
    try {
      await Auth.signUp({username, password});
      setUsername(username);
      setPassword(password);
      return {success: true, message: ''};
    } catch (error) {
      return {
        success: false,
        message: '認証に失敗しました。',
      };
    }
  };

  const confirmSignUp = async (verificationCode: string) => {
    try {
      await Auth.confirmSignUp(username, verificationCode);
      const result = await signIn(username, password);
      setPassword('');
      return result;
    } catch (error) {
      return {
        success: false,
        message: '認証に失敗しました。',
      };
    }
  };

  const signIn = async (username: string, password: string): Promise<Result> => {
    try {
      const result = await Auth.signIn(username, password);
      const hasChallenge = Object.prototype.hasOwnProperty.call(result, 'challengeName')
      setUsername(result.username);
      setIsAuthenticated(true);
      if (hasChallenge) {
        const challengeName = result.challengeName
        return {success: true, message: '', hasChallenge: true, challengeName};
      }
      return {success: true, message: '', hasChallenge: false}
    } catch (error) {
      return {
        success: false,
        message: '認証に失敗しました。',
      };
    }
  };

  const signInComplete = async (username: string, oldPassword: string, newPassword: string) => {
    try {
      const user = await Auth.signIn(username, oldPassword)
      await Auth.completeNewPassword(user, newPassword, user.challengeParam.requiredAttributes)
      return {
        success: true,
        message: ""
      }
    } catch (error) {
      if (error instanceof Error) {
        if (error.name === "InvalidParameterException" || error.name === "InvalidPasswordException") {
          return {
            success: false,
            message: 'パスワードは英字/数字組み合わせの8桁以上で入力してください。',
          };

        } else if (error.name === "NotAuthorizedException") {
          return {
            success: false,
            message: 'パスワードが間違っています。',
          };

        } else if (error.name === "ExpiredCodeException") {
          return {
            success: false,
            message: '仮パスワードの有効期限が切れています。カスタマーサポートへご連絡ください。',
          };

        } else if (error.name === "AuthError") {
          return {
            success: false,
            message: 'メールアドレスとパスワードを入力してください。',
          };

        } else if (error.name === "UserNotFoundException") {
          return {
            success: false,
            message: 'メールアドレスまたはパスワードが正しくありません。',
          };

        }
      }
      return {
        success: false,
        message: '予期せぬエラーが発生しました。',
      };
    }
  }

  const signOut = async () => {
    try {
      await Auth.signOut();
      setUsername('');
      setIsAuthenticated(false);
      return {success: true, message: ''};
    } catch (error) {
      return {
        success: false,
        message: 'ログアウトに失敗しました。',
      };
    }
  };

  const changePassword = async (user: any, oldPassword: string, newPassword: string, newPasswordConfirm: string) => {
    try {
      if (newPassword === newPasswordConfirm) {
        await Auth.changePassword(user, oldPassword, newPassword);
        return {
          success: true,
          message: '',
        };
      } else {
        return {
          success: false,
          message: '入力した新規パスワードが一致しません。もう一度入力してください。',
        };
      }
    } catch (error) {
      if (error instanceof Error) {
        if (error.name === "NotAuthorizedException") {
          return {
            success: false,
            message: '現在のパスワードが間違っています。',
          };
        } else if (error.name === "LimitExceededException") {
          return {
            success: false,
            message: 'しばらく時間を置いて再度試してください。',
          };
        } else if (error.name === "InvalidPasswordException") {
          return {
            success: false,
            message: 'パスワードは英字（小文字必須）/数字組み合わせの8桁以上で入力してください。',
          };
        } else if (error.name === "InvalidParameterException") {
          return {
            success: false,
            message: 'パスワードを入力してください。',
          };
        }
      }
      return {
        success: false,
        message: 'パスワードを入力してください。',
      };
    }
  }

  const forgetPassword = async (email: string) => {
    const regex = /^(([^<>()[\]\\.,;:\s@"]+(\.[^<>()[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    if (regex.test(email)) {

      try {
        await Auth.forgotPassword(email)
        return {
          success: true,
          message: '',
        }

      } catch (error) {
        return {
          success: false,
          message: '予期せぬエラーが発生しました',
        }

      }
    } else {
      return {
        success: false,
        message: 'メールアドレスの形式で入力してください',
      };
    }
  }

  const resetPassword = async (username: string, code: string, newPassword: string, newPasswordConfirm: string) => {
    if (newPassword !== newPasswordConfirm) {
      return {
        success: false,
        message: '入力した新規パスワードが一致しません。もう一度入力してください。'
      }
    }
    try {
      await Auth.forgotPasswordSubmit(username, code, newPassword);
      return {
        success: true,
        message: ""
      }

    } catch (error) {
      if (error instanceof Error) {
        if (error.name === "InvalidParameterException" || error.name === "InvalidPasswordException") {
          return {
            success: false,
            message: 'パスワードは英字（小文字必須）/数字組み合わせの8桁以上で入力してください。',
          };
        } else if (error.name === "AuthError" || error.name === "ExpiredCodeException" || error.name === "LimitExceededException") {
          if (error.message === "Confirmation code cannot be empty") {
            return {
              success: false,
              message: 'コードを入力してください。',
            };
          } else {
            return {
              success: false,
              message: 'コードの有効期限が切れています。再度プロフィール画面から「編集」をクリックしてください。',
            };
          }
        } else if (error.name === "CodeMismatchException") {
          return {
            success: false,
            message: 'コードが間違っています。',
          };
        }
      }
      return {
        success: false,
        message: "予期せぬエラーが発生しました"
      }
    }
  }

  return {
    isLoading,
    isAuthenticated,
    username,
    currentAuthenticatedUser: () => Auth.currentAuthenticatedUser(),
    signUp,
    confirmSignUp,
    signIn,
    signInComplete,
    signOut,
    changePassword,
    forgetPassword,
    resetPassword
  };
};

```
::::

変更点は、以下の通りです。

- `signInComplete`, `changePassword`, `forgetPassword`, `resetPassword`の関数の追加。
  - これによって、必要な処理を実装できます。
- `currentAuthenticatedUser`の追加。
  - userを取得することができます。

## 各種認証画面の作成

各種画面は、`AuthBasePage.tsx`と各種`*.form.tsx`をそれぞれ組み合わせて作成します。

`AuthBasePage.tsx`だけでは、以下の内容を表示するようになっており、
各種`*.form.tsx`がそれぞれの入力フォームになっています。

![](https://storage.googleapis.com/zenn-user-upload/a63c2667ad40-20230225.png =600x)
*`AuthBasePage.tsx`のみで表示される内容*

::::details コード詳細

以下のコードは、ほとんどこの[参考記事](https://qiita.com/shunnami/items/98a341d2ac20775241ad)の内容がベースになっています。

```typescript jsx: AuthBasePage.tsx
import React, {createContext, useContext, useState} from 'react';
import {Alert, Avatar, Grid, Paper, Snackbar, Typography} from "@mui/material";
import {teal} from "@mui/material/colors";
import LockOutlinedIcon from "@mui/icons-material/LockOutlined";

interface UseAuthPage {
  setIsError: (state: boolean) => void
  setErrorMessage: (state: string) => void
}

const authPageContext = createContext({} as UseAuthPage);

export const useAuthPage = () => {
  return useContext(authPageContext);
};

export const AuthBasePage: React.FC<{ title: string, children: React.ReactNode }> = ({title, children}) => {
  const [isError, setIsError] = useState(false);
  const [errorMessage, setErrorMessage] = useState('');

  const value = {
    setIsError,
    setErrorMessage
  }

  return (
    <>
      <Grid>
        <Paper
          elevation={3}
          sx={{
            p: 4,
            height: "70vh",
            width: "280px",
            m: "20px auto"
          }}
        >
          <Grid
            container
            direction="column"
            justifyContent="flex-start" //多分、デフォルトflex-startなので省略できる。
            alignItems="center"
          >
            <Avatar sx={{bgcolor: teal[400]}}>
              <LockOutlinedIcon/>
            </Avatar>
            <Typography variant={"h6"} sx={{m: "30px"}}>
              {title}
            </Typography>
          </Grid>
          <authPageContext.Provider value={value}>
            {children}
          </authPageContext.Provider>

        </Paper>
      </Grid>
      {/*メッセージ*/}
      <Snackbar open={isError} autoHideDuration={null}>
        <Alert onClose={() => setIsError(false)} severity="error" sx={{width: '100%'}}>
          {errorMessage}
        </Alert>
      </Snackbar>
    </>
  )
}

```

::::

![](https://storage.googleapis.com/zenn-user-upload/99f9a9e63896-20230225.png =600x)
*パスワード忘れ画面*

![](https://storage.googleapis.com/zenn-user-upload/ffb6b2886b5d-20230225.png =600x)
*パスワード再設定画面*

![](https://storage.googleapis.com/zenn-user-upload/6d972248aa69-20230225.png =600x)
*パスワード変更画面*


# おわりに

# 参考

- 一番参考にした記事：[Amplify を使わず React で AWS Cognito 認証を使う](https://mseeeen.msen.jp/react-auth-with-ready-made-cognito/)
- [【MUI】コピペで使える：ログインページ](https://qiita.com/shunnami/items/98a341d2ac20775241ad)
- [React 初心者が Material-UI で今どきの Web フォームを作ってみた（react-hook-form編）](https://dev.classmethod.jp/articles/react-beginners-tried-to-create-a-modern-web-form-with-material-ui-and-react-hook-form/)
- [react-hook-formの使い方まとめ](https://qiita.com/NozomuTsuruta/items/60d15d97eeef71993f06#seterror)