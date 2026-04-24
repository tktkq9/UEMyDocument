# Serialization 概要

- 上位: [[01_core_overview]]
- 関連: [[UObject/01_overview]] | [[Reflection/01_overview]]
- ソース: `Engine/Source/Runtime/Core/Public/Serialization/`, `Engine/Source/Runtime/CoreUObject/Public/Serialization/`, `CoreUObject/Public/UObject/Linker*.h`

---

## Serialization とは

UE5 における **バイナリ/テキスト I/O の統一抽象**。同じコードで以下のすべてに対応する:

- **アセット保存・ロード** — `.uasset`/`.umap` への変換
- **ネットワーク複製** — ビット単位のパケット生成
- **SaveGame** — プレイヤーのセーブデータ
- **エディタの Copy/Paste** — アクタ・プロパティのテキスト化
- **DDC（Derived Data Cache）** — コンパイル済みシェーダ等のキャッシュ

中核は `FArchive` 抽象クラス。`operator<<` を `IsLoading()` / `IsSaving()` で分岐させる **単方向インターフェース** が特徴。

---

## アーキテクチャ

```mermaid
graph TD
    subgraph レガシー[レガシー FArchive]
        AR[FArchive<br>operator&lt;&lt;]
        AR --> MR[FMemoryReader / FMemoryWriter]
        AR --> FR[FFileReader / FFileWriter]
        AR --> BR[FBitReader / FBitWriter]
        AR --> LR[FLinkerLoad / FLinkerSave]
    end
    subgraph 新方式[新方式 StructuredArchive / CompactBinary]
        SA[FStructuredArchive<br>スロットベース]
        SA --> BAF[BinaryArchiveFormatter]
        SA --> JAF[JsonArchiveFormatter]
        CB[FCbObject / FCbField<br>Compact Binary]
        CB --> DDC[DDC / Zen Storage]
    end
    LR -->|読み書き| PKG[.uasset / .umap]
    PKG --> UO[UObject Tree]
```

---

## FArchive の基本形

```cpp
void UMyObject::Serialize(FArchive& Ar)
{
    Super::Serialize(Ar);

    Ar << MyInt;           // int32 / float / FVector 等は組込
    Ar << MyString;        // FString / FName / FText
    Ar << MyArray;         // TArray / TMap / TSet
    Ar << ObjectRef;       // UObject* も自動で参照として処理

    if (Ar.IsLoading())
    {
        // ロード時のみの処理（例: バージョン互換）
    }
    if (Ar.IsSaving())
    {
        // セーブ時のみの処理（例: 派生データ計算）
    }
}
```

**UPROPERTY() がついたメンバは `UObject::Serialize()` が自動でリフレクション経由でシリアライズするため、通常は Super 呼び出しで十分**。カスタム処理が必要な場合のみオーバーライドする。

---

## 主要クラス

| クラス | 役割 |
|-------|------|
| `FArchive` | シリアライズ基底。`operator<<(T&)`・`IsLoading()`/`IsSaving()`・`ArVersion` |
| `FArchiveState` | `FArchive` の内部状態（バージョン・フラグ等）を別オブジェクトに分離 |
| `FArchiveProxy` | 別の `FArchive` にフォワードするプロキシ（ラッピング用）|
| `FMemoryArchive` / `FMemoryReader` / `FMemoryWriter` | メモリバッファへの I/O |
| `FBufferArchive` | 可変長バッファに書き込み |
| `FBitReader` / `FBitWriter` | ビット単位 I/O（ネットワーク複製で使用） |
| `FStructuredArchive` | スロットベースの構造化 I/O（Binary / JSON を切替可能） |
| `FCustomVersion` / `FCustomVersionContainer` | モジュール別のバージョニング（互換性維持） |
| `FCompactBinaryObject` (`FCbObject`) | UE5 新フォーマット（DDC / Zen で使用） |
| `FLinkerLoad` | `.uasset` ロードの中核（パッケージ → UObject ツリー復元） |
| `FLinkerSave` | `.uasset` セーブの中核 |
| `FPackageFileSummary` | `.uasset` のヘッダ情報 |
| `FBulkData` | 大容量バイナリデータ（メッシュ・テクスチャ）の遅延ロード |
| `IAsyncPackageLoader` / `FAsyncLoadingThread2` | 非同期パッケージロード（Zen Loader） |

---

## パッケージフォーマット

`.uasset` の構造:

```
+-----------------------------+
| FPackageFileSummary         | ← マジックナンバー、バージョン、オフセット
+-----------------------------+
| Name Table                  | ← 使用されている FName の配列
+-----------------------------+
| Import Table                | ← 他パッケージのオブジェクト参照
+-----------------------------+
| Export Table                | ← このパッケージが含む UObject の一覧
+-----------------------------+
| Export Data                 | ← 各 UObject::Serialize() が書いたデータ
+-----------------------------+
| BulkData / EditorOnly       | ← 大容量データ・エディタ専用データ
+-----------------------------+
```

**ロード手順** (`FLinkerLoad::Preload()`):
1. Summary 読み込み
2. Name/Import/Export テーブル展開
3. Export ごとに UObject を生成（クラスは Import から解決）
4. 各 UObject の `Serialize()` を呼び出してプロパティを復元

---

## 非同期ロード（Zen Loader / AsyncLoadingThread2）

UE5 では **EDL（Event Driven Loader）** が **Zen Loader / AsyncLoadingThread2** に置き換わりつつある。

- **AsyncLoadingThread** が別スレッドで動作し、ファイル I/O・UObject 生成・依存解決を並列化
- **GameThread との同期点** は `PostLoad()` と一部のコールバックのみ
- **バッチ化**: 複数パッケージの依存グラフを解析して I/O を最適化

CVar `s.AsyncLoadingThreadEnabled` で有効/無効切替。

---

## カスタムバージョン

互換性を維持するためのモジュール単位バージョニング:

```cpp
struct FMyCustomVersion
{
    enum Type { InitialVersion = 0, AddedFeatureX, LatestVersion = AddedFeatureX };
    static const FGuid GUID;
};

// 登録
FCustomVersionRegistration GMyCustomVersion(FMyCustomVersion::GUID,
    FMyCustomVersion::LatestVersion, TEXT("MyCustomVersion"));

// 使用
void UMyObject::Serialize(FArchive& Ar)
{
    Ar.UsingCustomVersion(FMyCustomVersion::GUID);
    if (Ar.CustomVer(FMyCustomVersion::GUID) >= FMyCustomVersion::AddedFeatureX)
    {
        Ar << NewMember;
    }
}
```

エンジン本体の例: `FFortniteMainBranchObjectVersion` / `FUE5MainStreamObjectVersion` 等多数。

---

## Details（個別記事）

| ドキュメント | 内容 |
|------------|------|
| [[Details/a_farchive]] | `FArchive` の基本・`operator<<`・IsLoading/IsSaving・カスタムバージョン |
| [[Details/b_asset_serialization]] | `.uasset` のロード/セーブ・`FLinkerLoad`/`FLinkerSave`・パッケージ構造 |
| [[Details/c_save_game]] | `USaveGame`・`UGameplayStatics::SaveGameToSlot`・バイナリ/JSON |

---

## Reference

- [[Reference/ref_serialization_api]] … `FArchive` / `FStructuredArchive` / `FLinkerLoad` の API

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `s.AsyncLoadingThreadEnabled` | `1` | 非同期ロードスレッド有効化 |
| `s.LevelStreamingActorsUpdateTimeLimit` | `5.0` | 非同期ロード時間制限（ms/フレーム） |
| `s.UseBackgroundLevelStreaming` | `1` | バックグラウンドロード有効化 |
| `s.EnableBulkDataLoading` | `1` | BulkData 遅延ロード有効化 |

---

## 備考

- **FArchive は単方向** — Load/Save を同じコードで書くため、`operator<<` しか使えない（`>>` なし）
- **FStructuredArchive は UE 4.22 以降** — 従来の `FArchive` と互換性あり（`FStructuredArchiveFromArchive` でラップ可）
- **CompactBinary は DDC 専用** — UE5.1+ で DDC が Compact Binary ベースに移行
- **Zen Loader は Windows/Console のみ** — モバイルは引き続き EDL
