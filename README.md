# central-parent（中央ビルド親pom）

複数の CAP Java アプリ（consumer）が `<parent>` 座標参照だけで**依存ライブラリの版を継承**するための中央親pom。
依存版を一元管理し、**親pom を固定版で publish するだけで、子は pom 無変更（版指定なし）で新しい依存を取り込める**ことを目的とする。

- 座標：`com.example.central:central-parent`
- publish 先：GitHub Packages（`https://maven.pkg.github.com/miyasuta/cap-java-parent-pom-sample`）
- 現行版：`2.1.0`（emoji-plugin 1.2.0 を pin）

> 「事前検証済みライブラリの版上げを手動確認なしで全 consumer に自動展開する」昇格モデルの**上流**。仕組み全体は consumer 側ドキュメント [cap-java-child / GUIDE-promotion-model.md](https://github.com/miyasuta/cap-java-child/blob/main/docs/guide/GUIDE-promotion-model.md) を参照。

## 管理しているもの

### 依存（`dependencyManagement`）— 子は版指定なしで使える

| 依存 | 種別 | 版の管理 |
|---|---|---|
| `com.example.cap.plugins:emoji-plugin` | 検証対象の共有ライブラリ（プラグイン） | `${emoji-plugin.version}` = **1.2.0** |
| `com.sap.cds:cds-services-bom` | CAP（CDS）の BOM | `${cds.services.version}` = 4.8.0 |
| `org.springframework.boot:spring-boot-dependencies` | Spring Boot の BOM | `${spring.boot.version}` = 3.5.11 |

### ツール／プラグイン版（`pluginManagement`・`properties`）

- JDK：`${jdk.version}` = 21
- cds-maven-plugin / spring-boot-maven-plugin / maven-compiler-plugin / surefire / flatten / enforcer の版を一括固定（子は version 指定なしで execution だけ定義）。

> **emoji-plugin が「検証対象」**＝この親pom の版上げが、全 consumer への自動ロールアウトの起点になる。CAP/Spring の BOM は基盤の版固定。

## 子側での使用方法

### 1. 親として座標参照する（`relativePath` 空 → レジストリ解決）

```xml
<parent>
  <groupId>com.example.central</groupId>
  <artifactId>central-parent</artifactId>
  <version>2.1.0</version>
  <relativePath/>
</parent>
```

### 2. emoji-plugin を**版指定なし**で依存に追加

```xml
<dependency>
  <groupId>com.example.cap.plugins</groupId>
  <artifactId>emoji-plugin</artifactId>
</dependency>
```

版は親の `dependencyManagement` が供給するため、子は書かない。CAP/Spring の依存も同様に版指定なしで利用できる。

### 3. GitHub Packages の取得先・認証は `settings.xml` 側に置く

親pom には `<repositories>` を**あえて書かない**（子は「親pom 自身」を読む前に取得先を知る必要があるため、pom では解決をブートストラップできない）。`~/.m2/settings.xml`（または `-s` 指定）にワイルドカードで定義する：

```xml
<repository>
  <id>github</id>
  <url>https://maven.pkg.github.com/miyasuta/*</url>
  <releases><enabled>true</enabled></releases>
  <snapshots><enabled>true</enabled></snapshots>
</repository>
```

`<server id=github>` に `read:packages` 権限の PAT を設定（詳細は [GUIDE-central-build-setup.md](https://github.com/miyasuta/cap-java-child/blob/main/docs/guide/GUIDE-central-build-setup.md)）。

## リリース手順（版を上げて publish）

emoji-plugin の新版が出たら：

1. `pom.xml` の `<emoji-plugin.version>` を新版に上げる
2. **親自身の `<version>` も上げる**（例 2.1.0 → 2.2.0）
   - 上げ忘れると同一版の再 publish になり GitHub Packages が **409 Conflict** を返す
3. publish：

```bash
mvn -N deploy
```

publish 後、各 consumer の Renovate が `<parent>` の新版を検知し、PR を自動作成する。

## 関連リポジトリ

| リポジトリ | 役割 |
|---|---|
| **cap-java-parent-pom-sample**（本リポジトリ） | 親pom（`central-parent`） |
| [cap-java-emoji-plugin](https://github.com/miyasuta/cap-java-emoji-plugin) | 検証対象の共有ライブラリ（`emoji-plugin`） |
| [cap-java-child](https://github.com/miyasuta/cap-java-child) | consumer（昇格モデルの下流・ドキュメント） |
