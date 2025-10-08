---
type: note
tags:
- KEN-90
- '#ai/ref'
---
> [!info] 必要なセクションを選択して使用してください！

# 🔗 前提条件・依存関係

- 今まではfrontendでcognitoユーザーを作成し、そのidをbackendに送ってユーザーを自社DBに登録していました
- しかし、backendでエラーが出た場合、cognitoユーザーを削除することができずトランザクションが担保されていませんでした 
- そこでバックエンドの処理が失敗した場合、cognitoのユーザーを削除し、トランザクションを担保できるようにすることとしました。

# ✅ 要件

## 1. Cognitoクライアントクラスの実装
- `src/infrastructures/clients/aws`配下にcognitoのクライアントクラスを実装し、deleteUserメソッドを持つ
## 2. ManageUser Adapterクラスの実装
- `src/domains/generatedSafetyInstructions/adapters/generateSafetyInstructions.ts`を参考に`src/domains/user`配下にユーザーの削除処理のインターフェースを定義する
- `src/infrastructures/adapters/api`にインターフェースの実装をする
- 実装には1.で作成したCognitoクライアントクラスを使用する
## 3. ManageUserAPIをusecase / route層で使用
- `src/usecases/users/createUserFromExternalMemberInvitation.ts`, `src/usecases/users/createUserFromInternalMemberInvitation.ts`, `src/usecases/users/createUserFromOwnerInvitation.ts`で2.で作成したManageUserAPIを使用する
	- usecase内で処理が失敗した場合、cognitoのユーザーを削除する
- 同様にroute層も修正する
## 4. Invitationの期限を7日から30日に変更する
- 変更する
## 5. Invitationの取得処理の修正
- `src/usecases/externalMemberInvitations/getExternalMemberInvitation.ts`, `src/usecases/internalMemberInvitations/getInternalMemberInvitation.ts`, `src/usecases/ownerInvitations/getOwnerInvitation.ts`に「①：より新しいInvitationが存在する場合はステータスをInvalidに変更してエラーを返す」「②：対象のユーザーが存在する場合はステータスをInvalidに変更してエラーを返す」「③：expiredの場合はエラーを返す」「④：acceptedの場合はエラーを返す」「invalidの場合はエラーを返す」というドメインロジックを実装
	- ステータス：Invalidは存在しないのでドメインモデルに追加する必要あり。

