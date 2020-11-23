# Kubernetes Security

本章ではコンテナのオーケストレーションツールである Kubernetes への攻撃例を紹介します。  
Kubernetes では多数のコンポーネントが存在しているため Attack Surfaces を理解する必要があります。

その理解を助けるために Microsoft が作成している ATT&CK ライクな Kubernetes attack matrix が参考になります。[^1]

![Kubernetes attack matrix](https://www.microsoft.com/security/blog/wp-content/uploads/2020/04/k8s-matrix.png)

この図を見ると一口に Kubernetes セキュリティと言ってもクラスタへのアクセス制御や Pod / コンテナのセキュリティ、ネットワークなど多岐にわたります。  
本章で全てを紹介することはできませんが、Pod に侵害された場合の権限昇格に焦点を当てて、いくつかの攻撃例と対策を紹介します。

---

[^1]: https://www.microsoft.com/security/blog/2020/04/02/attack-matrix-kubernetes/
