# ExcelTest

GitHub Copilot CLI がデータ処理スクリプトを実行する際に、Python ではなく **C# file-based apps** (`dotnet run -`) を使わせるための **Copilot Skill** の実装例です。

このリポジトリでは、そのスキルを使って Excel ファイル（`.xlsx`）を解析・変換できることを実際に試すことができます。

## 仕組み

`.github/skills/scripting-guide/SKILL.md` に定義されたスキルが、GitHub Copilot CLI に以下のルールを適用します。

- コードは **パイプで `dotnet run -` に渡して実行** する（`.cs` ファイルの新規作成は禁止）
- 外部ライブラリは `#:package` ディレクティブで指定（例: `#:package ClosedXML`）
- Python 等のスクリプト言語の代わりに C# を使用する

## 前提条件

- [.NET 10 SDK](https://dotnet.microsoft.com/download) 以降
- [GitHub Copilot CLI](https://githubnext.com/projects/copilot-cli)

## 試し方

1. リポジトリをクローンする
2. GitHub Copilot CLI で Excel ファイルに対する操作を依頼する

```
@Employee.xlsx を markdown にして```

Copilot は Python ではなく、C# file-based apps を使って処理を実行します。

### 実行例

```powershell
# Excel の内容をダンプ
@'
#:package ClosedXML@0.104.2

using ClosedXML.Excel;

using var workbook = new XLWorkbook(args[0]);
var worksheet = workbook.Worksheet(1);

foreach (var row in worksheet.RowsUsed())
{
    Console.WriteLine(row.Cell(1).Value);
}
'@ | dotnet run - -- Employee.xlsx
```

## リポジトリ構成

```
.github/
  skills/
    scripting-guide/
      SKILL.md          # Copilot Skill 定義
Employee.xlsx             # サンプル Excel ファイル
Employee.md               # Excel から変換された Markdown
```

## ライセンス

[MIT](./LICENSE)
