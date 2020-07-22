# Clientの利用

controller-runtimeでは、Kubernetes APIにアクセスするためのクライアントとして[client.Client](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client?tab=doc#Client)を提供しています。

[import:"init"](../../codes/tenant/main.go)
[import:"new-manager",unindent:"true"](../../codes/tenant/main.go)


最初に、`runtime.NewScheme()`で新しい`scheme`を作成し、

`clientgoscheme.AddToScheme`では、PodやServiceなどKubernetesが標準リソースの型をschemeに追加しています。
`multitenancyv1.AddToScheme`では、先程作成したTenantカスタムリソースの型をschemeに追加しています。

このschemeを利用してクライアントを作成することで、

`ctr.GetConfigOrDie()`でクライアントの設定を取得しています。
この関数はコマンドラインオプションの`--kubeconfig`や、環境変数`KUBECONFIG`で指定された設定ファイルを利用するか、
またはKubernetesクラスタ上でPodとして動いているのであれば、Podが持つサービスアカウントの認証情報を利用します。
通常、コントローラはKubernetesクラスタ上で動きますので、クラスタから割り当てられた設定を利用すれば問題ありません。

これらの設定を用いてmanagerを作成したら、`GetClient()`でクライアントを取得することができます。
なお後述するように、このクライアントは`Get()`や`List()`でリソースを取得すると、同一namespaceの同じKindのリソースをすべて取得してインメモリにキャッシュします。
このようなキャッシュの仕組みが必要ない場合は、`GetAPIReader()`でキャッシュを利用しないクライアントを取得することもできます。
基本的には`GetClient()`で取得するクライアントを利用すれば問題ありません。

## Get/List

### Getの使い方

[import:"get",unindent="true"](../../codes/tenant/controllers/tenant_controller.go)

### キャッシュ

### インデックス

index field: リソースごとに一意になっていればよい。 実態のフィールドの構成と一致していなくても良い。
informerはgvkごとに作られる。namespaceは自動的にキーに付与されるので、わざわざつけなくてもよい。
戻り値がスライスになっている、複数の値でインデクシングすることも可能。


リソース一覧を取得する際に、条件でフィルタリングしたいことがあるかと思います。
ループで回してもいいのですが、

インメモリキャッシュにインデックスを張ることができます。
インデックスを利用するためには事前に`GetFieldIndexer().IndexField()`を利用して、TenantリソースのConditionReadyの値に応じてインデックスを作成しておきます。

[import:"indexer"](../../codes/tenant/controllers/tenant_controller.go)
[import:"index-field",unindent:"true"](../../codes/tenant/controllers/tenant_controller.go)

上記のようなインデックスを作成しておくと、`List()`を呼び出す際に特定のフィールドが指定した値と一致するリソースだけを取得することができます。
例えば以下の例であれば、ConditionReadyが"True"のTenantリソース一覧を取得することが可能です。

[import:"matching-fields",unindent:"true"](../../codes/tenant/controllers/tenant_controller.go)

フィールド名には、どのフィールドを利用してインデックスを張っているのかを示す文字列を指定します。
実際にインデックスに利用しているフィールドのパスと一致していなくても問題はないのですが、なるべく一致させたほうが可読性がよくなるのでおすすめです。
なおinformerはGVKごとに作成されるので、異なるタイプのリソース間でフィールド名が同じになっても問題ありません。
またnamespaceスコープのリソースの場合は、自動的にフィールド名にnamespace名が付与されるので、明示的にフィールド名にnamespaceを含める必要はありません。

## Create/Update

[import:"namespace,create",unindent:"true"](../../codes/tenant/controllers/tenant_controller.go)


## CreateOrUpdate

Createはリソースがすでに存在していた場合には失敗
Updateはリソースが存在しない場合には失敗

CreateOrUpdateを利用すると、リソースが存在しなければ作成し、存在すれば更新してくれます。

[import:"create-or-update",unindent:"true"](../../codes/tenant/controllers/tenant_controller.go)

## Patch

SSA

## Status.Update/Patch

statusをサブリソース化していない場合、この処理は失敗するので注意


## Delete/DeleteOfAll