---
title: "プライベート エンドポイントの名前解決の挙動を見てみる"
emoji: "🚫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure","Networking","DNS"]
published: false
---

この記事では、Azure プライベート エンドポイント利用時の名前解決の挙動について、実際の検証手順と結果をまとめます。  
「どのように名前解決されるのか？」を知りたい方の参考になれば幸いです。

# 🔧準備

仮想マシン (Windows Server 2022 Datacenter) を２台デプロイし、ストレージ アカウントを用意します。  
- 1台目: DNS サーバー用途 (パブリック IP なし)  
- 2台目: クライアント用途  
- ストレージ アカウント: 最初はネットワークアクセス制限なし

# 🧪実験1:パブリック エンドポイントへの名前解決の確認

現時点、クライアント VM からストレージ アカウント (Blob) へのアクセスはパブリック エンドポイント経由になります。
その点を確認するため、Blob のエンドポイント FQDN に対して名前解決を確認してみます。

![alt text](/images/pe-behavior/azpe15.png)

ちなみにストレージ アカウントのエンドポイントは、Azure ポータルの "エンドポイント" から一覧確認できます。

![alt text](/images/pe-behavior/azpe1.png)

それではクライアント VM から Resolve-DnsName コマンドで名前解決します。
見ると、CNAME レコードで blob.tyo22prdstr08a.store.core.windows.net が得られており、
さらに A レコードでパブリック IP アドレスに解決されています
(CNAME に設定されているのは、ストレージノードの FQDN 名になります)。

```powershell
PS C:\Users\azureuser> Resolve-DnsName saforpelab.blob.core.windows.net

Name                           Type   TTL   Section    NameHost
----                           ----   ---   -------    --------
saforpelab.blob.core.windows.n CNAME  60    Answer     blob.tyo22prdstr08a.store.core.windows.net
et

Name       : blob.tyo22prdstr08a.store.core.windows.net
QueryType  : A
TTL        : 23
Section    : Answer
IP4Address : 20.150.85.228
```

# 🧪実験2: ストレージ アカウントでプライベート エンドポイントを有効化

次にストレージ　アカウントで、Blob サービスのプライベート エンドポイントを有効化します。

![alt text](/images/pe-behavior/azpe14.png)

またこの際、実験3 で自前の DNS サーバーを利用するため、プライベート DNS ゾーンは作成しません。

![alt text](/images/pe-behavior/azpe2.png)
![alt text](azpe3.png)

それでは再度みてみましょう。
すると、最終的な結果 (ストレージ サーバーの FQDN 名が得られ、パブリック IP アドレスを取得) は同じであるものの、
その前に "saforpelab.privatelink.blob.core.windows.net" という CNAME レコードが追加されていることがわかります。

プライベート DNS ゾーンがある場合は、"saforpelab.privatelink.blob.core.windows.net" に対する A レコードが登録されることで、
プライベート IP アドレスに解決されることになります。

```powershell
PS C:\Users\azureuser> Resolve-DnsName saforpelab.blob.core.windows.net

Name                           Type   TTL   Section    NameHost
----                           ----   ---   -------    --------
saforpelab.blob.core.windows.n CNAME  60    Answer     saforpelab.privatelink.blob.core.windows.net
et
saforpelab.privatelink.blob.co CNAME  60    Answer     blob.tyo22prdstr08a.store.core.windows.net
re.windows.net

Name       : blob.tyo22prdstr08a.store.core.windows.net
QueryType  : A
TTL        : 52
Section    : Answer
IP4Address : 20.150.85.228
```

# 🧪実験3: カスタム DNS サーバーを利用した名前解決

最後に、カスタム DNS サーバーによるプライベート エンドポイントの名前解決を試してみましょう。

![alt text](/images/pe-behavior/azpe13.png)

DNS サーバーのインストール手順は、[こちら](https://learn.microsoft.com/ja-jp/windows-server/networking/dns/quickstart-install-configure-dns-server?tabs=gui) を参照してください。

以下の２点の設定を行います。

1) privatelink.blob.core.windows.net のゾーンを作成し、A レコードにプライベート エンドポイントの IP アドレスを登録
2) blob.core.windows.net に対しては、条件付きフォワーダーを設定し、Azure の規定 DNS サーバー (168.63.129.16) へフォワード

## 1) privatelink.blob.core.windows.net のゾーンを作成し、A レコードにプライベート エンドポイントの IP アドレスを登録

Server Manager から "DNS" を選択し、DNS サーバーを右クリックして、
"DNS Manager" を選択します。

![alt text](/images/pe-behavior/azpe4.png)

"Forward Lookup Zones" を右クリックし、"New Zone..." を選択します。

![alt text](/images/pe-behavior/azpe5.png)

"privatelink.blob.core.windows.net" のゾーンを作成します。

![alt text](/images/pe-behavior/azpe6.png)

作成したゾーンに対し、次いで A レコードを追加します。この時、ストレージ アカウント名とプライベート エンドポイントの IP アドレスを指定します。

![alt text](/images/pe-behavior/azpe7.png)
![alt text](/images/pe-behavior/azpe8.png)

## 2) blob.core.windows.net に対しては、条件付きフォワーダーを設定し、Azure の規定 DNS サーバー (168.63.129.16) へフォワード

同様に "DNS Manager" を開き、"Conditional Forwarders" を右クリックし、"New Conditional Forwarder..." を選択します。
そして、こちらで "blob.core.windows.net" を指定し、Azure の規定 DNS サーバー (168.63.129.16) へフォワードを設定します。

![alt text](/images/pe-behavior/azpe10.png)

クライアント VM 側のネットワーク インターフェースで、カスタム DNS として DNS サーバー (10.0.0.5) を設定し、
クライアント VM を再起動し反映させます。

![alt text](/images/pe-behavior/azpe9.png)

これで、Blob ストレージに対するプライベート エンドポイントの名前解決ができるようになりました。
試してみましょう。

期待通り、CNAME "saforpelab.privatelink.blob.core.windows.net" が得られ、
それがプライベート エンドポイントのプライベート IP アドレス 10.0.0.6 に解決されることが確認できました。

```powershell
PS C:\Users\azureuser> Resolve-DnsName saforpelab.blob.core.windows.net

Name                           Type   TTL   Section    NameHost
----                           ----   ---   -------    --------
saforpelab.blob.core.windows.n CNAME  59    Answer     saforpelab.privatelink.blob.core.windows.net
et

Name       : saforpelab.privatelink.blob.core.windows.net
QueryType  : A
TTL        : 3600
Section    : Answer
IP4Address : 10.0.0.6
```

最後にクライアント VM から Blob へ書き込みしてみます。
具体的には、ストレージ アカウント (Blob) に対してファイルをアップロード操作を実行します。
なお、認証には VM のマネージド ID を有効化し、それに対して "ストレージ BLOB データ共同作成者" を割り当てておきます。
また、トラフィックの確認のため、パケットキャプチャを実行します。
Network Watcher を使ったパケットキャプチャについては、[Azure の機能を使って Azure VM のネットワークキャプチャを取る](https://qiita.com/aktsmm/items/6d77b8af1eb10f28cab5) にて分かりやすく解説されてます。

```powershell
PS C:\Users\azureuser> Clear-DnsClientCache
PS C:\Users\azureuser> $timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
PS C:\Users\azureuser> $filePath = "C:\Temp\timestamp.txt"
PS C:\Users\azureuser> $timestamp | Out-File -FilePath $filePath -Encoding UTF8
PS C:\Users\azureuser> azcopy login --identity
INFO: Login with identity succeeded.
PS C:\Users\azureuser> $storageAccount = "saforpelab"
PS C:\Users\azureuser> $container = "test"
PS C:\Users\azureuser> azcopy copy $filePath "https://$storageAccount.blob.core.windows.net/$container/timestamp.txt"
INFO: Scanning...

...(省略)...

100.0 %, 1 Done, 0 Failed, 0 Pending, 0 Skipped, 1 Total, 2-sec Throughput (Mb/s): 0.0001


Job 9ef5305b-eef9-664a-74b9-96c6c9ae542a summary
Elapsed Time (Minutes): 0.0333
Number of File Transfers: 1
Number of Folder Property Transfers: 0
Number of Symlink Transfers: 0
Total Number of Transfers: 1
Number of File Transfers Completed: 1
Number of Folder Transfers Completed: 0
Number of File Transfers Failed: 0
Number of Folder Transfers Failed: 0
Number of File Transfers Skipped: 0
Number of Folder Transfers Skipped: 0
Number of Symbolic Links Skipped: 0
Number of Hardlinks Converted: 0
Number of Special Files Skipped: 0
Total Number of Bytes Transferred: 24
Final Job Status: Completed
```

上記で Blob データをアップロードできました。
それでは、取得したパケットキャプチャを確認してみましょう。

始めに DNS クエリが DNS サーバー (10.0.0.5) に送られ、期待通りプライベート エンドポイントのプライベート IP アドレス (10.0.0.6) に解決されることが確認できました。

![alt text](/images/pe-behavior/azpe11.png)

次に、Blob ストレージに対し SSL ハンドシェイクが行われ、実データの通信がその後に続いていることがわかります。

![alt text](/images/pe-behavior/azpe12.png)

という感じで、プライベート エンドポイント経由で Blob ストレージへのアクセスができてますね。

# まとめ

今回は、あえてカスタム DNS サーバーを利用してプライベート エンドポイントの名前解決を行う方法を見てみましたが、
SLA 100% ですし、プライベート DNS ゾーンを利用する方が管理も容易でお薦めです。

## 参考

* [Azure プライベート DNS ゾーンに関するベスト プラクティス及びよくあるお問合せ](https://jpaztech.github.io/blog/network/private-dns-zone-faq/)
* [Azure プライベート エンドポイントの DNS 統合シナリオ](https://learn.microsoft.com/ja-jp/azure/private-link/private-endpoint-dns-integration)