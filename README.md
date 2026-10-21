# ゲームアップデートに伴うモジュールパケット仕様変更の原因分析と移行ガイド
**— 効果量取得失敗（ALL +1化）・ツールクラッシュの技術的真相と修正手順 —**

> **対象読者**: 本ゲームのネットワークパケットを解析・利用する外部ツール（モジュールシミュレータ、DPSメーター、インベントリ同期ツール等）の開発者、および改修を担当するAIコーディングエージェント  
> **更新日**: 2026年10月  
> **ステータス**: 実パケット（400KB生バイナリ）検証完了・修正検証済み

---

## 1. 概要と発生している現象

2026年10月のゲームアップデート以降、モジュール関連のパケットを処理する各種外部ツールにおいて、以下の障害が多発しています。

1. **効果量がすべて「+1」または「0」になる**:
   * モジュールの効果名（筋力、HP変動、精鋭など）は取得できるが、効果量（リンク数）がすべて `+1` として記録・表示される。
2. **ツールの異常終了（クラッシュ）**:
   * Pythonツール: `KeyError: <module_uuid>`
   * C# / .NETツール: `NullReferenceException: Object reference not set to an instance of an object`
   * Rust / Goツール: `Option::unwrap()` パニック、または空マップ参照。
3. **ゲーム画面上の表示枠逆転現象**:
   * ゲーム内UIにおいて、「本来1枠目にしか出現しない極効果（極・HP変動など）」が3段目（3枠目）に表示される表示バグが同時発生している。

---

## 2. 根本原因の技術的解説 (Root Cause Analysis)

### 2.1 変更前（旧仕様）のデータ構造
従来、モジュールの効果名と効果量は**2つの異なるコンテナに分離して格納**されていました。

```text
SyncContainerData (MethodID: 0x15)
└── VData (CharSerialize)
    ├── ItemPackage (Tag 7)
    │   └── Packages[5] (モジュール格納庫)
    │       └── Items[uuid] (Item)
    │           ├── Uuid (Tag 1)
    │           ├── ConfigId (Tag 2)
    │           └── ModNewAttr (Tag 13)
    │               └── ModParts (Tag 1) = [部位ID1, 部位ID2, 部位ID3] (例: [2404, 1114, 1113])
    │
    └── Mod (Tag 57)
        ├── ModSlots (Tag 1) = 装備中スロット情報
        └── ModInfos (Tag 2) = Map<int64, ModInfo> ★ここに全モジュールの効果量があった
            └── ModInfo (key = モジュールUUID)
                └── InitLinkNums (Tag 4) = [効果量1, 効果量2, 効果量3] (例: [10, 5, 1])
```

各ツールは `ModParts[i]` の効果名に対し、`ModInfos[uuid].InitLinkNums[i]` をインデックスで紐付けて効果量を取得していました。

---

### 2.2 変更後（新仕様）のデータ構造
アップデートにより、サーバー側のProtobuf構造に大幅なリファクタリングが実施されました。

1. **`Mod.ModInfos (Tag 2)` は完全廃止（空・0件）**:
   * `CharSerialize.Mod (Tag 57)` の中身は、現在装備中モジュールのスロット情報（`ModSlots (Tag 1)`）のみ（約37バイト）となり、**`ModInfos` は完全に送信されなくなりました**。
   * これにより、`ModInfos[uuid]` を参照していた既存コードは例外（`KeyError` / `NullReference`）を吐いてクラッシュするか、例外処理で空配列となりフォールバック（初期値 `+1`）が発動していました。
2. **効果量は各 `Item` 直下の新設フィールド `Tag 21` へ移動**:
   * 外部コンテナ（`ModInfos`）への参照が廃止され、インベントリ内の各 `Item` 自身が自身の効果量を直接保持する形式に変更されました。
   * **格納場所**: `Item.Tag 21`

```text
SyncContainerData (MethodID: 0x15)
└── VData (CharSerialize)
    ├── ItemPackage (Tag 7)
    │   └── Packages[5]
    │       └── Items[uuid] (Item)
    │           ├── Uuid (Tag 1)
    │           ├── ConfigId (Tag 2)
    │           ├── ModNewAttr (Tag 13)
    │           │   └── ModParts (Tag 1) = [2404, 1114, 1113] (スロット順)
    │           │
    │           └── ★【新設】LinkAttrs (Tag 21)
    │               └── Values (Tag 1) = Map<int32, int32>
    │                   ├── [Key: 2404 (HP変動)] => Value: 10
    │                   ├── [Key: 1114 (精鋭)]   => Value: 5
    │                   └── [Key: 1113 (特殊)]   => Value: 1
    │
    └── Mod (Tag 57)
        ├── ModSlots (Tag 1) (装備枠 5件のみ保持)
        └── ModInfos (Tag 2) = 【廃止・空（0件）】
```

---

### 2.3 【最重要】Tag 21 の Protobuf Wire 型定義の注意点

AIエージェントや開発者が最も陥りやすい落とし穴が、**`Tag 21` の Protobuf Wire レベルの階層構造**です。

生パケットのバイナリダンプ（HEX/デシリアライズ結果）は以下の通りです：

```text
Item Tag 21 Raw Bytes:
[10, 5, 8, 228, 18, 16, 10,  10, 5, 8, 218, 8, 16, 5,  10, 5, 8, 217, 8, 16, 1]

分解:
先頭バイト 0x0A (10) = (Field 1 << 3) | WireType 2 (Length-Delimited, len=5)
  ├── 0x08 (8)  = Field 1 (Varint) => 2404 (0x964 = HP変動)
  └── 0x10 (16) = Field 2 (Varint) => 10
次バイト 0x0A (10) = (Field 1 << 3) | WireType 2 (Length-Delimited, len=5)
  ├── 0x08 (8)  = Field 1 (Varint) => 1114 (0x45A = 精鋭)
  └── 0x10 (16) = Field 2 (Varint) => 5
次バイト 0x0A (10) = (Field 1 << 3) | WireType 2 (Length-Delimited, len=5)
  ├── 0x08 (8)  = Field 1 (Varint) => 1113 (0x459 = 特殊)
  └── 0x10 (16) = Field 2 (Varint) => 1
```

> **[!CAUTION] スキーマ定義の重大な注意**  
> `Item` の直下に `map<int32, int32> LinkAttrs = 21;` と定義すると、Protobufデシリアライザは `Item` 直下の Tag 21 に `(key=1, value=2)` のエントリが並ぶことを想定するため、**`UnexpectedWireType: expected Varint, got LengthDelimited` でデコードエラーを起こします**。  
> 
> 正しくは、**「Tag 21 はサブメッセージであり、そのサブメッセージの Tag 1 が Map である」** という構造です。

---

## 3. ゲーム内表示バグ（極効果が3段目に表示される理由）の真相

ユーザーから報告されている「ゲーム画面で極効果が本来の1枠目ではなく3段目に表示される」挙動も、このパケット構造の解析によって証明されました。

* **サーバーから届くパケット**:
  * `ModParts`（Tag 13）: `[HP変動(極), 精鋭, 特殊]` （1枠目＝極の正しい順序）
  * `LinkAttrs`（Tag 21）: `HP変動=>10, 精鋭=>5, 特殊=>1` （正しいマッピング）
* **ゲームクライアント側のバグ**:
  * 今回のアップデートで、**ゲームクライアント側のUI描画ルーチンがモジュールのリストを下から上へ逆順描画（逆順イテレート）してしまうバグ**を抱えています。
  * したがって、**パケットデータ自体のスロット順序（ModParts）は完全に正常**です。ツール側で無理にスロット順を逆転させる必要はありません。

---

## 4. 実装移行ガイド (Code Migration Guide)

### 4.1 Protobufスキーマ (`.proto`) の改修

`.proto` ファイルの `Item` 定義に、Tag 21 のメッセージを追加します。

```protobuf
syntax = "proto3";
package zproto;

message Item {
    int64 Uuid = 1;
    int32 ConfigId = 2;
    int64 Count = 3;
    int32 Quality = 9;
    ModNewAttr ModNewAttr = 13;
    
    // ★新設: モジュール効果量マップ（Tag 21）
    ModLinkAttr LinkAttrs = 21;
}

message ModLinkAttr {
    // Tag 1 に PartId => 効果量 のマップが格納される
    map<int32, int32> Values = 1;
}

message ModNewAttr {
    repeated int32 ModParts = 1;
}
```

---

### 4.2 Python実装例 (`module_parser.py` 等)

```python
def extract_module_effects(item, mod_infos=None):
    """
    モジュールの効果名と効果量を抽出する関数
    新仕様 (Item.Tag 21) を最優先し、旧仕様 (ModInfos) にもフォールバック対応
    """
    if not item.HasField('ModNewAttr') or not item.ModNewAttr.ModParts:
        return []

    mod_parts = list(item.ModNewAttr.ModParts)
    effects = []

    # 1. 新仕様: Item 直下の Tag 21 (LinkAttrs) をチェック
    # (コンパイル済み protobuf に LinkAttrs がある場合)
    if hasattr(item, 'LinkAttrs') and item.HasField('LinkAttrs') and item.LinkAttrs.Values:
        val_map = item.LinkAttrs.Values  # {part_id: link_num}
        for part_id in mod_parts:
            val = val_map.get(part_id, 1)
            effects.append((part_id, val))
        return effects

    # 2. 旧仕様フォールバック: Mod.ModInfos[uuid].InitLinkNums
    if mod_infos and item.Uuid in mod_infos:
        mod_info = mod_infos[item.Uuid]
        link_nums = list(mod_info.InitLinkNums)
        for i, part_id in enumerate(mod_parts):
            val = link_nums[i] if i < len(link_nums) else 1
            effects.append((part_id, val))
        return effects

    # 3. 最終安全策: 部位IDの出現回数 (すべて +1)
    for part_id in mod_parts:
        effects.append((part_id, 1))

    return effects
```

---

### 4.3 C#実装例 (`BPSR-ZDPS` / `MessageManager.cs` 等)

```csharp
public static List<(int PartId, int Value)> ExtractModuleEffects(Item item, Mod modData)
{
    var results = new List<(int, int)>();
    if (item.ModNewAttr == null || item.ModNewAttr.ModParts.Count == 0)
        return results;

    var parts = item.ModNewAttr.ModParts;

    // 1. 新仕様: Item.LinkAttrs (Tag 21) から取得
    if (item.LinkAttrs != null && item.LinkAttrs.Values != null && item.LinkAttrs.Values.Count > 0)
    {
        var valMap = item.LinkAttrs.Values; // IDictionary<int, int>
        foreach (var partId in parts)
        {
            int val = valMap.TryGetValue(partId, out var v) ? v : 1;
            results.Add((partId, val));
        }
        return results;
    }

    // 2. 旧仕様フォールバック: Mod.ModInfos
    if (modData?.ModInfos != null && modData.ModInfos.TryGetValue(item.Uuid, out var modInfo))
    {
        var linkNums = modInfo.InitLinkNums;
        for (int i = 0; i < parts.Count; i++)
        {
            int val = i < linkNums.Count ? linkNums[i] : 1;
            results.Add((parts[i], val));
        }
        return results;
    }

    // 3. セーフティフォールバック
    foreach (var partId in parts)
    {
        results.Add((partId, 1));
    }
    return results;
}
```

---

### 4.4 Rust実装例 (`prost` 使用)

```rust
pub fn extract_module_effects(
    item: &Item,
    mod_infos: Option<&HashMap<i64, ModInfo>>,
) -> Vec<(i32, u8)> {
    let mut effects = Vec::new();
    let parts = match &item.mod_new_attr {
        Some(attr) if !attr.mod_parts.is_empty() => &attr.mod_parts,
        _ => return effects,
    };

    // 1. 新仕様: Item.link_attrs (Tag 21)
    if let Some(link_attr) = &item.link_attrs {
        if !link_attr.values.is_empty() {
            for &p in parts {
                let val = link_attr.values.get(&p).copied().unwrap_or(1);
                effects.push((p, val as u8));
            }
            return effects;
        }
    }

    // 2. 旧仕様フォールバック: ModInfos
    if let Some(infos) = mod_infos {
        if let Some(mod_info) = infos.get(&item.uuid) {
            let link_nums = &mod_info.init_link_nums;
            for (i, &p) in parts.iter().enumerate() {
                let val = if i < link_nums.len() { link_nums[i] } else { 1 };
                effects.push((p, val as u8));
            }
            return effects;
        }
    }

    // 3. セーフティフォールバック
    for &p in parts {
        effects.push((p, 1));
    }
    effects
}
```

---

## 5. テスト用実測データ（回帰テスト・検証用）

修正後の動作確認用に、今回の生パケットから抽出された代表的なモジュール実測データを記載します。

| モジュールUUID | 種別 ConfigId | ModParts 部位構成 | 新仕様 Tag 21 格納値 (PartId => Value) | 期待される抽出結果 |
| :--- | :--- | :--- | :--- | :--- |
| **14856** | 5500103 | `[2404, 1114, 1113]` | `2404=>10, 1114=>5, 1113=>1` | HP変動 +10, 精鋭 +5, 特殊 +1 |
| **21975** | 5500303 | `[2404, 1409, 1407]` | `2404=>10, 1409=>2, 1407=>4` | HP変動 +10, 会心 +2, 詠唱速度 +4 |
| **21123** | 5500103 | `[2404, 1110, 1112]` | `2404=>9, 1110=>5, 1112=>3` | HP変動 +9, 筋力 +5, 知力 +3 |
| **23067** | 5500303 | `[2404, 1408, 1407]` | `2404=>6, 1408=>3, 1407=>5` | HP変動 +6, 攻撃速度 +3, 詠唱速度 +5 |
| **18966** | 5500203 | `[2404, 1410, 1205]` | `2404=>6, 1410=>7, 1205=>1` | HP変動 +6, 幸運 +7, 特攻回復 +1 |

---

## 6. まとめ

* アップデートによる「効果量取得失敗(クラッシュ及び+1化など)」の直接原因は、**`Mod.ModInfos (Tag 57-2)` の廃止と、各 `Item` 直下の新フィールド `Tag 21 (LinkAttrs)` への移動**です。
* `Tag 21` は `Item` 直下の Map ではなく、**`ModLinkAttr` メッセージ内部に `map<int32, int32> Values = 1` を持つ構造**です。
* 各言語（Python / C# / Rust）ともに、`Item.LinkAttrs` を最優先で読み、旧パケット用に `ModInfos` にフォールバックする実装を行うことで、過去の保存データとの互換性を保ちながら完全復旧できます。
