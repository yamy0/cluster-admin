# kubeadmでのkubernetesクラスタ構築
kubeadmを用いて、VirtualBox上のVMにKubernetesクラスタを構築する。  
マスターノード1台、ワーカーノード2台の構成にする。  
各ノードは互いに通信可能なネットワーク構成にする。  

## ノード(仮想マシン)の構築
### VirtualBox NAT Network
ゲストPCの仮想ネットワークをNATする。  
ゲストPCを互いに通信可能な同一NW上に構築するため、NAT Networkを用いる。
* 環境設定ウィンドウを開く  
[ファイル]-[環境設定]
* 左ペイン[ネットワーク]を開く
* 右端「+」マークのついたボタンを押下する
* NatNetworkという名称のネットワークが作成される。
* [OK]を押下して環境設定ウィンドウを閉じる。

### 仮想HW構成
仮想マシンを新規作成する。
* ゲストOSはubuntu20.04
* メモリは4096MB
* ディスクは10GB
  * argocdデプロイ時にディスク領域不足になったので、32GBに増やした。  
* ネットワークアダプタを1つ構成し、作成済みのNatNetworkに接続する。

### OSインストール
Ubuntuをインストーラの指示にしたがってインストールする。
* キーボードレイアウトの設定だけ、どうしても英語配列になってしまったので、インストール後に変更。
* インストーラのオプションで、opensshをインストールしておく。

### ログイン情報
* Master
  * User: master
* Worker
  * User: worker
### キーボードレイアウト変更
```
sudo dpkg-reconfigure keyboard-configuration
Generic 105-key PC (intl.)
Japanese
Japanese
The default for the keyboard layout
No compose key
```
### IPアドレス固定
netplanに設定を読み込ませる。  

設定ファイル：/etc/netplan/99_config.yaml
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:         # ここの名称は環境によって変わる
      dhcp4: false
      dhcp6: false
      addresses: [10.0.2.5/24]
      gateway4: 10.0.2.1
      nameservers:
        addresses: [10.0.2.1]
```
反映コマンド
```
sudo netplan apply
```
### レガシーiptablesを利用する
nftablesがデフォルトで有効になっている。  
kubeadmはnftablesをサポートしていない。そのため、iptablesを導入、有効化する。  
```
sudo apt-get install -y iptables arptables ebtables
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```

* その他iptables設定
ブリッジを通過するトラフィックを処理できるようにする。  
```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### swap無効化
現在のswap領域を確認する。  
```
cat /proc/swaps
```
今回は、有効な領域がないため、スキップする。

### kubeadm kubelet kubectlのインストール
kubeletおよびkubectlはkubeadmの管理対象外であるため、個別にインストールする。  
その際、kubeadmでインストールされるコンポーネントとバージョン差異が発生しないようにする。
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### コンテナランタイムのインストール
今回はDockerを利用する。
ubuntuインストール時にsnapで入れたDockerは、systemd管理のユニット仕様が独特で、kubeadmとの相性が悪い。  
そのためsnapですでに入れた場合は、
```
snap remove docker
```
でアンインストールする

```
## パッケージインデックスの更新
# apt-get update

## 前提パッケージを導入
# apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

## GPGKEYの取得と登録
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

## リポジトリ登録
# add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

## インストール
# apt-get install -y docker-ce
```
### 仮想マシンのクローン
マスターノード用に作成、設定した仮想マシンをワーカーノード用にクローンする。
以降はマスターノード、ワーカーノードで実施内容が異なるため、ここでクローンする。

### ワーカーノードの作成
VirtualBoxで作業を実施する。
* まずは、マスターノードを停止する。
* 仮想マシンのクローンウィンドウを開く。  
[仮想マシン]-[クローン]
* 仮想マシンのクローンウィンドウで、[エキスパートモード]を選択し、以下を設定する。
  * 名前: 任意の名前とする
  * クローンのタイプ: すべてをクローン
  * スナップショット: 現在のマシンの状態
  * 追加オプション:
    * MACアドレスのポリシー: すべてのネットワークアダプターでMACアドレスを生成
    * チェックボックス: 何もチェックしないこと

### 重複設定の排除
* 作成したワーカーノードを起動する。
* ホスト名を変更する。
```
hostnamectl set-hostname [ホスト名]
```
* ユーザ名を作成する(必須でない)
変更対象ユーザのセッションは閉じる。
別のユーザでログインする。
```
sudo usermod -l [変更後ユーザ名] [変更前ユーザ名]
sudo usermod -d /home/worker -m worker
```
* IPアドレスを変更する。
```
sudo vi /etc/netplan/99_config.yaml
※network.ethernets[*].enp0s3.addressesの値を変更する。
sudo netplan apply
```

## kubeadmによるkubernetesクラスタ構築
### コントロールプレーンノードの初期化

```
# kubeadm init --control-plane-endpoint=10.0.2.5:6443 --pod-network-cidr=192.168.0.0/16
```
* --control-plane-endpoint  
今回は、シングルノードでのコントロールプレーンだが、冗長化する場合に必要となるため念のため指定しておく。  
コントロールプレーン同士で、他のコントロールプレーンのエンドポイントを認識するためのオプション。  

* --pod-network-cidr  
Pod同士の通信に必要なCNIの実装ごとに、使用するネットワークを指定する。  
CIDR表記で指定するが、どのネットワークかは、CNIごとに決まっている。  
今回はCalicoを使用する。  

### calico設定
* calicoのマニフェストを取得する。
```
curl https://docs.projectcalico.org/manifests/calico-etcd.yaml -o calico.yaml
```
* etcdをTLS通信で動かしている場合に必要な設定
マニフェストを編集して、configmapとsecretの設定を変更する。
変更する項目は以下の通り
  * calico-etcd-secrets
    * data.etcd-key
      * 設定する値は以下のコマンドの結果
      ```
      cat /etc/kubernetes/pki/etcd/server.key | base64 -w 0
      ```
    * data.etcd-cert
      * 設定する値は以下のコマンドの結果
      ```
      cat /etc/kubernetes/pki/etcd/server.crt | base64 -w 0
      ```
    * data.etcd-ca
      * 設定する値は以下のコマンドの結果
      ```
      cat /etc/kubernetes/pki/etcd/ca.crt | base64 -w 0
      ```
  * calico-config
    * etcd_endpoints
      * 設定する値は以下のコマンドの結果で表示されるetcdプロセスの--advertise-client-urlsオプションの値
      ```
      ps -ef | grep etcd
      ```
    * etcd_ca
      * デフォルトの""を削除して、コメントアウトされている値を有効化する。
    * etcd_cert
      * デフォルトの""を削除して、コメントアウトされている値を有効化する。
    * etcd_key
      * デフォルトの""を削除して、コメントアウトされている値を有効化する。

編集後の該当箇所の内容は以下の通り
```
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # The keys below should be uncommented and the values populated with the base64
  # encoded contents of each file that would be associated with the TLS data.
  # Example command for encoding a file contents: cat <file> | base64 -w 0
  etcd-key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBeHR1YStrcWxhcGFZdW5zbEpJbEZVTWt0VVU2SkNSK1RiVm9yY0V4Y1FOcEZabUxyClNJeERkV2xCSm01TStqRzE1djZpZ01abUF4R2RXVkc4ZzlJVEYxTU9aYTNyZmwxQmx6TXZ1bW5TblZ1TFNJakcKWkk3N0cvaTlsNFVISHdCQTZzWUtBclpxNkZOUzd4SWJYcGVqU1NqbHRYdDhlQXF4REdyZVN1UnI0V2VtYVVyawpraStmTEtYbFVjSDRBYnU2eFZybGgzUk5tK0tpK1kzSS80NHVlL0R3T2xKdUZxOGRsZE02anl6VzRUcXFiaXN0CkZIVTF5eVpyMzZNWTQ0ZW9Eak5HSm5XQ2dGTEk2QnI3SXl1KzJSbzhWUWlRR1RQYlpnUlZMVy8wdlRVTFBqbzIKMEFXcFYrOC9TQkpCYjBNazdmWFhrSWxSeVkwOHlKVFNpWmFtTVFJREFRQUJBb0lCQVFERU1sOVJtdHhoc1h1MgpZVkZnSHQ5NHVVUXcrQjlVRlFkTDJLOEsrUXY4SUY1Z2lqQkJQOEkrMFQ0cVFLRktLRW1rUW83cUp0VDNLaVhvCnZqQVVqdXV2RTQ4YzJ0K0JxVmpSYVBzcUhNWmo0cklsT20walFiNlc5bTk0VXhPWGpwUitEaTVLUnRocnArb24KWGVJTERlbFlnVFZDUFRlczZEK09WTkpGYWVEV1hjSnlFOFFHVmZUMmJKUjdFeWQrSkVDbmNWdDQ1NVA5UVhpWApnNW5mcS9XM0NjaXF3MnhOL1k0ZXJFMVoyRTBYUlZRNDVFN0VJRUhQUkJlb0VrT0lhK2lpYUx0eDhxZ1FJVndBClpmZzduZEl4NkdOZXAzRnh5RkIrODB5NVhSQzFwODZKcWZYajF2L2JvOEpVR3FTRDA0WDNyRnZ2TTRrUzI4c2cKTS93THpGakJBb0dCQU03QmRyaVd1amlHcXR3eEZUZG9ublNvVks3VkxqeGlaUkhJNWFvaG8vN2FRdmhvb0ZsYwp6SUh3NFBhbUJuanplNXVpS1VuczR4NzFuS0wwZDJVOFU4Yms3MEtqdVhxRnJQd0xXSkRCRWQrZzNXbEs1OElMCjF0YTk1d01UeEN3SW5oak0vTkwwZWpsTVpVMWFWNkpGQy93bHZGMXFTRFdNK3hLNnFNMEU3akoxQW9HQkFQWTQKbFVxMmczK1c4Z3QwL2FMTjNncnkzL2pzb3Jsa0dlcFl3U01WRVJOUnpTTEVHMzVkOWpLUktDQXd4Y3lNdEpOdgpxVnlTblZ0MDFQZEJrV3dmd1BMVFpNM2NlUUNhbjh3d3M2OVlmdk5KTEREWUVyVzQ0aTRLZGc0UU1aQnBQSlVJCklUZm5WdWlaYTlUcWx1aFdJV1NCWk1hMHFPODhML3djdDRtdEduVk5Bb0dBQnRyNVVjT0ZweXduN0NjZ2VmYloKRWlzbXE2bGI0QnF2R1RqZERKZ1M5UGROc3lqYzhEbVllbEovVXc1TU5xUjBHOFB6dElUTFB4S0x3QWQxRWdFLwpFZUF6WXJWRkNCLzRqVjdlNytYRzd2QkpoeDA1dEFCcWZqSkx2NWxmTHNxV1cySW9tK0lKVDI4T0NOT1BCazFkCnlWMkM4bUg4eFBISXZXVTlCWmM5UXFVQ2dZRUF6QkpOdW1UWFRIS2hIbG5TdHBNR1MvRE5MWldEc1VDRU1qVnAKcmxnUmxQK2hsQVVSL0lTSVA1VUx1dEp4dm4ySVZRS2hUbmErTVVUK0ZnaWtMUWVNZGpZN1FGeFJkZXl5TVJ6VQpjS3BhWGUzeDBISGwzL1Bpa3VKY3duOHRkVkdqd3FuQVRvTlJCdXZSOGVDVlB1L1VNV2NGVGFRQ3VIWWNGMHI5CjNBQTdBNmtDZ1lCT1Vlc0JYMzFZMjFKSjdsQ3FjTlpZcFgwajNqNTNsTWo1SWNnN280aC9pOUZLK1MvYUVPb3YKOEpra21reVB5SnpuTjR3dUovVzNDaXdMRFc4TG1NMUsxMnZxTGNBYWp0VnJBOXZJVWg3TDFvQllreXRobExHSQplZDlnZktwcnZ6dmNyTFh1RjEyRjNyVllwdW5ubmRIU0hNM0NYb0g4VURIRVY2Wkd2SGJBYWc9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
  etcd-cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURPVENDQWlHZ0F3SUJBZ0lJZDlERGNEYVRETXd3RFFZSktvWklodmNOQVFFTEJRQXdFakVRTUE0R0ExVUUKQXhNSFpYUmpaQzFqWVRBZUZ3MHlNREV3TURNd05qVXdNVEJhRncweU1URXdNRE13TmpVd01UQmFNQk14RVRBUApCZ05WQkFNVENHMWhjM1JsY2pBeE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBCnh0dWEra3FsYXBhWXVuc2xKSWxGVU1rdFVVNkpDUitUYlZvcmNFeGNRTnBGWm1MclNJeERkV2xCSm01TStqRzEKNXY2aWdNWm1BeEdkV1ZHOGc5SVRGMU1PWmEzcmZsMUJsek12dW1uU25WdUxTSWpHWkk3N0cvaTlsNFVISHdCQQo2c1lLQXJacTZGTlM3eEliWHBlalNTamx0WHQ4ZUFxeERHcmVTdVJyNFdlbWFVcmtraStmTEtYbFVjSDRBYnU2CnhWcmxoM1JObStLaStZM0kvNDR1ZS9Ed09sSnVGcThkbGRNNmp5elc0VHFxYmlzdEZIVTF5eVpyMzZNWTQ0ZW8KRGpOR0puV0NnRkxJNkJyN0l5dSsyUm84VlFpUUdUUGJaZ1JWTFcvMHZUVUxQam8yMEFXcFYrOC9TQkpCYjBNawo3ZlhYa0lsUnlZMDh5SlRTaVphbU1RSURBUUFCbzRHUk1JR09NQTRHQTFVZER3RUIvd1FFQXdJRm9EQWRCZ05WCkhTVUVGakFVQmdnckJnRUZCUWNEQVFZSUt3WUJCUVVIQXdJd0h3WURWUjBqQkJnd0ZvQVVFbEZCc2psVXNpcmgKbm9JeXZCYUthL29zRndzd1BBWURWUjBSQkRVd000SUpiRzlqWVd4b2IzTjBnZ2h0WVhOMFpYSXdNWWNFQ2dBQwpCWWNFZndBQUFZY1FBQUFBQUFBQUFBQUFBQUFBQUFBQUFUQU5CZ2txaGtpRzl3MEJBUXNGQUFPQ0FRRUF4b254CnN3QXlVM1daSWprdG5nOHFRMUNRa25KOERidFhMSWMwclQzT1g5SkkyNlRFeWU5NjhQTWRJU3dFVWRJQjNWRmkKYTJtazJTYkZuRHhFU3ZLRzZZTTBnMmVpenFOSW4vVng3R1VhR0dHYk02ZnNaRFE1RWlsZmNJNzdzRGE3K3FLNAp0VEhkUFUyYTlSRGcybjE0MHVsdW1QeElYZnd5b3ErQlZNajhJZHZ5OVhqOVAzRG9EMytqMCs1NVA4RnRJd2FzCk1CVGticTRmTERpdzJveUNLUmdTVTJzMThYOW9RdUNKaVhnUElFK29sWWo3NHM3dHVDQ2ROT0xVclVUZGY4RzUKL1ZiMlBiRmRBRnpFMWl2cjlqdlNIa3lyVWRJczE3WkZLNHI4ckFKMXByYlkvTUpTY1FJaTdNZnVKd3MwQ3FRcApwbXhxb0tzUnNGUW1jdHdiOGc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  etcd-ca: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM0VENDQWNtZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFTTVJBd0RnWURWUVFERXdkbGRHTmsKTFdOaE1CNFhEVEl3TVRBd016QTJOVEF4TUZvWERUTXdNVEF3TVRBMk5UQXhNRm93RWpFUU1BNEdBMVVFQXhNSApaWFJqWkMxallUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU1nOHh2MUdidEZJCnk4bUFncWw2N2pNemFWVlR4cHBlZ1F2ZThHbFlOOEk1Mk9UbHBxOVpFZ2t0alkrQ1BQUVArUGx2eDBoK09KQ2wKRGdUMUpZS2VDajBaUnV5VG0wZXR4YWJ2VFhDY2s2MU5hd2pvR2pTbnhzcytyTDlGT1NEbmlwU3VrWjlpY05PUApUWmVXK01LbFJSam1TVDFITzd3Z2ZOcnU5OUREcVY5YmZlM0JVbFhKK1puMDc4UTQyVE51MnBHRWo3S2txNklYClpTcENQK2M0ckVCcmJLNllhQ2VFYVVvejlpWmNiekhBRnRTRllCWTFoTnVDOFVaMkFrT0hkb2ZSYjB4VmQzam8KdDlFWVQ3bWltMDFnQkoxOFpvTHc5MEV1L1docTNVZkNkYTBEc2twS1gyekVuWnFnQmtneXhIRzY0UVJURUhrbQpTd3FsaFFiWGwrVUNBd0VBQWFOQ01FQXdEZ1lEVlIwUEFRSC9CQVFEQWdLa01BOEdBMVVkRXdFQi93UUZNQU1CCkFmOHdIUVlEVlIwT0JCWUVGQkpSUWJJNVZMSXE0WjZDTXJ3V2ltdjZMQmNMTUEwR0NTcUdTSWIzRFFFQkN3VUEKQTRJQkFRQ3pvT1VmaklOKzhRNVFpOW5QQmlNekRxM2NkZlZCMXZRNjR3L0NQekxoM1d4WFArM2tvc2wxOVA0awpFWGZsUEUrMDhhckUyQkpmYjBGV2s3QnZMTHpLazdxTGQzemxEYTB6OEVmaTBJTUZOdHk0V0orcUdURjdmODlDCmZiUUltTHV6RFJkVnZ0U0FLeTgybFVnVUs4T0o0SU1sOGxISncwb0c3VUQ3VTNoZ2swTUtpYW5tNGlaTm5ubzUKbWdyUUZMQ3VQNkNDb3BKYks5eWFjN2NGMHRnWTZSckhNZUtoRG1vSFZWN3Q4ck13bE5CMExDWFFWdGh5M0pJdQpubmw4M2RlYzJudXRWY1UxbUFWbTArNG81UlMxTFh1SzUrbUhkdHkwc0JIbjNWa3NFZGh2YjJySllWRGZVNTJtCmlZWUE2N1BLSnluYkZ1SkYwNDh6M3orM2hTMHgKLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
---
# Source: calico/templates/calico-config.yaml
# This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # Configure this with the location of your etcd cluster.
  etcd_endpoints: "https://10.0.2.5:2379"
  # If you're using TLS enabled etcd uncomment the following.
  # You must also populate the Secret below with these files.
  etcd_ca: "/calico-secrets/etcd-ca"
  etcd_cert: "/calico-secrets/etcd-cert"
  etcd_key: "/calico-secrets/etcd-key"
```

### ワーカーノードのクラスタへの参加
kubeadm init実行時に出力されるjoin用コマンドをそのままワーカーノードで実行する。    

```
kubeadm join 10.0.2.5:6443 --token sqojx7.8f7v838aafdgqssi  --discovery-token-ca-cert-hash sha256:65d7d86d4f1f46b9a8b41b2dcc0c373022a03b120be22724999c075bdf21fcd7
```

## kubectl認証設定
### 接続ユーザ作成
必要に応じて、kubectlを利用するOSユーザを作成する。  
今回はmasterユーザでkubectlを利用する。

### PKI証明書作成
kubeapi-serverとのTLS通信で使用するクライアント証明書を生成する。  
#### 秘密鍵の生成
```
openssl genrsa -out master.key 2048
```
#### CSR(証明書署名要求)の生成
```
openssl req -new -key master.key -subj "/CN=master" -out master.csr
```
#### kubernetesのCSRリソースを作成する。  
※すでにkubectlできる環境が必要。(コントロールプレーンノードのrootやパブリッククラウドの場合はコンソールービス)  

CSRリソースのマニフェストを作成する。  
```
vi master-csr.yaml
(以下内容)
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: master
spec:
  groups:
  - system:authenticated
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0diV0Z6ZEdWeU1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTRJYkN2S0NpbUp6OXQ5bXZ5RkJ0SXRBTzlGSUZ5clArR0ltaHVYczMzTVJxCk5CY0RQcFRRN3hoWlNEMnRhWERtUWMvUTBleURaTGZFMm5TaEx4U1oxSnRrZk54THVTQXRCVUpDSm9QbGJVQ1gKN01DOWcvN3ZETkdOTkpUc3M1Z0RIcW9mZ2FBTEI3bHdwd1BPekpyZzJlZ2UzNHJ4UVdwOXZzTG0zU2I5c1BvMApNcGJnang1OHdPcWRFNVVkV01JMmtUSkRtQVorcHdBK2FWWDdndVZtRXl5SEJqSEtXWHllSDdPZkZrNW8yeGtYCmlzbEVKS2s2M0xxejZOQmpzY0hNRnlhVFpEazNZbVhpOHRnS2dwNklhSlRVYzBmNVdweTQ0bGJsdmlidFZlQjQKQWFxckZzMzRqcGdGMEdPSm8xK1E3MHBhbmR3SVI5V2M3R1FXemQ5c09RSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBS2x0alN4SjBTb0lpeS94SU9VaXZoSG54SWx2ZVpHQnRvMGk0SHZiNXFJOHVtRlYvWjd1CnlLWHVQQ2t5aFNZdnZBL0lHMHNsbVZSZ3FCU21kNWx2MmRQYnNGSC8za1NHdjFIRG9BWGQyR3U0TSsxOXZpaUwKWTl4K1NhRjgzMUp6N2x2UmxLMlZweWdTOENaWWljaytZYkxEUUFkcU9ISCs4VWVPSzVuRkJCTTdKbXFqL0ZldQpMTitrYUJmSUxzNFNQaDhOWExXUjNCaHplVk15ZEUySXVyQVFwSzNYTDRRM0ZiMHRkZmtkRzZ1M0dIZldXVi9JCjZWYlg0Wmlxb01hWndzUVBvUlVsRXNYWEJjQlVnY0dMNGw1YmpOUU9HblNNV0Z5eUxiSWhzdzFzMW1MTHJ1RDQKK2d0c1JwZk04b0l1UUx0RWJtR2pkWFRKZXhWTTVJWk5UcmM9Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  usages:
  - client auth
```
※requestの値は以下のように取得する。
```
cat master.csr | base64 | tr -d "\n"
```
CSRリソースを適用する。
```
kubectl apply -f master-csr.yaml
```

#### 証明書に署名する
```
kubectl certificate approve master
```
※master はCSRリソースの名称。

#### クライアント証明書を取得する。
```  
kubectl get csr master -o yaml
```
status.certificateの値が証明書。  

#### ロール設定
kubernetesにおけるユーザ権限を設定する。
```
kubectl create clusterrolebinding master-user --clusterrole=cluster-admin --user=master
```
master-userという名称のロールバインディングを生成する。  
ここでは、cluster-adminロールをmasterユーザにバインド(権限付与)する。  
ここでのユーザ名はクライアント証明書に記載されたコモンネーム。  

#### 各種証明書ファイルを作成する。  
  * クライアントキーファイル(作成済み)
  * クライアント証明書ファイル(ここではmaster.crtとして作成する)
  * CA証明書ファイル(ここではca.crtとして作成する)
  ```
  echo [証明書データ] | base64 --decode > [ファイル名]
  ```
  CA証明書データは、既存のkubectl環境のkubeconfigなどから取得する。  
  コントロールプレーンノードにSSHログインできる場合は、以下の通り。
  * /etc/kubernetes/pki/ca.crt(kubeadmで構築した環境のデフォルト)  
  →このファイルを直接使用してもいい。

### kubeconfig設定  

クラスタ設定
```
kubectl config set-cluster master01 --server=https://10.0.2.5:6443 --certificate-authority=/home/master/ca.crt --embed-certs=true
```
ユーザ設定
```
kubectl config set-credentials master --client-key=/home/master/master.key --client-certificate=/home/master/master.crt --embed-certs=true
```
ここで指定しているユーザ名masterはコンテキスト設定で指定する--userの値と同じ。  
証明書のコモンネームおよびロールにバインドしたユーザ名とは関係ないため、別でも動作に問題はない。  

コンテキスト設定
```
kubectl config set-context frommaster-tomaster01 --cluster=master01 --user=master
kubectl config use-context frommaster-tomaster01
```