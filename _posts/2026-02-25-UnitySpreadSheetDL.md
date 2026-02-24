---
layout: post
title:  "OAuth不要でGoogleスプレッドシートのデータをUnityエディタに直接ダウンロードする"
date:   2026-02-25
categories: Unity CI/CD
key: 2026-02-25-UnitySpreadSheetDL
---
### 1. はじめに

- **【動作環境】**
    - <u>Unity 2022.3.62f3</u>
- **やったこと:**
    - Unityエディタのメニューからボタン一つで、Googleスプレッドシートのデータを直接ダウンロードできるエディタ拡張を実装した。
- **仕組み:**
    - スプレッドシートのアクセス権を「リンクを知っている全員（閲覧者）」に設定し、エクスポート用のURL（`/export?format=tsv`）を生成する。
    - それをUnity側から `UnityWebRequest` を用いて叩くことで、複雑なOAuth認証などを省略し、データを取得している。
- **記事の趣旨（スコープ）:**
    - 本記事では、実際のコード例を交えながら「データのダウンロード部分」に焦点を絞って解説する。取得したデータをUnity内でどう展開・パースするかについては、プロジェクトに大きく依存するため今回は割愛する。


### 2. 手順

- **1. スプレッドシート側の設定**
    - **1-1. 共有設定の変更**
        - 画面右上の「共有」ボタンから、一般的なアクセスを **「リンクを知っている全員」（閲覧者）** に変更する。これにより、OAuth認証などの複雑なプログラムを省略し、直接データにアクセスできるようになる。
    - **1-2. 必要な2つのIDの取得**
        - プログラム側がシートにアクセスできるように、スプレッドシートのURLから以下の2カ所をコピーしておく必要がある。
            
            ```
            https://docs.google.com/spreadsheets/d/【①Spreadsheet ID】/edit#gid=【②Gid（シートID）】
            ```
        - ①スプレッドシートのID
        - ②シートのID
            
<br>
 
- **2. プログラム上の手順：設定情報の管理とURL生成**
    
    Unity側で各IDを保持し、ダウンロード用URLへ変換する手順。
    
    - **2-1. 「シート設定」保存用のScriptableObject**
        - コード例
            
            ```csharp
            using UnityEngine;
            using System.Collections.Generic;
            
            [CreateAssetMenu(fileName = "ScenarioImportSettings", menuName = "Scenario /Scenario Import Settings")]
            public class ScenarioImportSettings : ScriptableObject
            {
                [Header("Google Sheets Settings")]
                public List<SheetDefinition> Sheets = new List<SheetDefinition>();
            
                [System.Serializable]
                public class SheetDefinition
                {
                    [Tooltip("ログ表示や仮想ファイル名として使用する識別名（例: MainData）")]
                    public string SheetName;
                    
                    [Tooltip("スプレッドシートのURLに含まれるID部分 (/d/xxxxx/のxxxxx)")]
                    public string SpreadsheetId;
                    
                    [Tooltip("シート(タブ)固有のID (URL末尾の gid=yyyyy のyyyyy)")]
                    public string Gid;
                    
                    [Tooltip("チェックを外すとインポート対象から除外されます")]
                    public bool IsEnabled = true;
            
                    // ダウンロード用URLを生成
                    public string GetDownloadUrl()
                    {
                        if (string.IsNullOrEmpty(SpreadsheetId) || string.IsNullOrEmpty(Gid)) return "";
                        return $"https://docs.google.com/spreadsheets/d/{SpreadsheetId}/export?format=tsv&gid={Gid}";
                    }
                }
            }
            ```
            
            - **役割と仕組み**: スプレッドシートごとの情報をScriptableObjectで管理する設計。
            - **解説**: 単なるスプレッドシートのURLではなく、エクスポート用のURLを組み立てているのがポイント。URLに `/export?format=tsv` を付与することで、Google Sheets APIを使わずとも、直接TSV（またはCSV）形式のテキストデータとしてダウンロード可能になる。
            
            {: .caption-text }
            ▼scriptable objectの例。シナリオインポート設定で「本編」と「おまけ」の2シート分のデータがある（それぞれスプレッドシートのリンクから対応するIDを入力した状態）<br>
            ![scriptable objectの例](/assets/img/20260225/2-1.png){: .figure-image }
            
            <br>

        - **2-2. 「シート設定」の検索メソッド**
            - コード例
            
            ```csharp
            using UnityEngine;
            using UnityEditor; // AssetDatabase に必要
            
            // 中略
            
            private static ScenarioImportSettings LoadSettings()
            {
                string[] guids = AssetDatabase.FindAssets("t:ScenarioImportSettings");
                if (guids.Length == 0)
                {
                    Debug.LogError("Error: 'ScenarioImportSettings' asset not found in project. Please create one.");
                    return null;
                }
            
                string path = AssetDatabase.GUIDToAssetPath(guids[0]);
                var settings = AssetDatabase.LoadAssetAtPath<ScenarioImportSettings>(path);
                if (settings == null)
                {
                    Debug.LogError($"Error: Failed to load settings at {path}");
                }
                return settings;
            }
            ```
            
            - **役割と仕組み**: エディタ上で設定ファイル（`ScenarioImportSettings`）を読み込む処理。
            - **解説**: `AssetDatabase.FindAssets("t:ScenarioImportSettings")` を使用している。パスを直接指定（ハードコード）せず、型（`t:`）で検索させることで、プロジェクト内のどこにアセットを配置しても自動でロードできる。この手の設定ファイルを複数作ることもあまりないのでこれで事足りる（と思う）。

            <br>
            <br>
            

- **3. プログラム上の手順：通信とデータ取得の全体フロー**
    - **3-1. 全体の起点**
        - コード例
        
        ```csharp
        using UnityEngine;
        using UnityEditor; // MenuItem, EditorUtility に必要
        using System.Collections.Generic; // Dictionary に必要
        using System.Linq; // IEnumerableの拡張メソッド(Where, ToListなど)に必要
        
        // 中略
        
        [MenuItem("Tools/Import Scenario/From Google Sheets")]
        public static async void ImportFromGoogleSheets()
        {
            if (!EditorUtility.DisplayDialog("Import from Google Sheets", 
                "Are you sure you want to import scenario data from Google Sheets?\nThis will overwrite existing ScriptableObjects.", 
                "Import", "Cancel"))
            {
                return;
            }
        
            Debug.Log("[TSVImporter] Import from Google Sheets started...");
        
            // 1. 設定ファイルのロード
            var settings = LoadSettings();
            if (settings == null) return;
        
            // 2. 有効なシートのリストアップ
            var targetSheets = settings.Sheets.Where(s => s.IsEnabled).ToList();
            if (targetSheets.Count == 0)
            {
                Debug.LogWarning("No enabled sheets found in settings.");
                return;
            }
        
            // 3. ダウンロード実行
            var tsvContents = new Dictionary<string, string>();
            bool hasDownloadError = false;
        
            foreach (var sheet in targetSheets)
            {
                string url = sheet.GetDownloadUrl();
                if (string.IsNullOrEmpty(url))
                {
                    Debug.LogError($"Invalid URL settings for sheet: {sheet.SheetName}");
                    hasDownloadError = true;
                    continue;
                }
        
                Debug.Log($"Downloading: {sheet.SheetName}...");
                try
                {
                    string content = await FetchText(url);
                    
                    // 仮想的なファイル名としてシート名を使用（拡張子.tsvを付与）
                    string virtualFileName = $"{sheet.SheetName}.tsv";
                    
                    if (tsvContents.ContainsKey(virtualFileName))
                    {
                        Debug.LogError($"Duplicate SheetName detected: {sheet.SheetName}. Please rename in settings.");
                        hasDownloadError = true;
                        continue;
                    }
        
                    tsvContents.Add(virtualFileName, content);
                }
                catch (System.Exception e)
                {
                    Debug.LogError($"Failed to download {sheet.SheetName}: {e.Message}");
                    hasDownloadError = true;
                }
            }
        
            if (hasDownloadError)
            {
                Debug.LogError("Import aborted due to download errors.");
                return;
            }
        
            // 4. TSVデータの展開
            ProcessImport(tsvContents);
        }
        ```
        
        - **役割と仕組み**: `[MenuItem]` 属性により、Unityのメニューバーから呼び出されるインポート処理の起点。
        - **解説**: 処理は主に以下のフローで進行する。
            1. `LoadSettings` で設定ファイルを取得する。
            2. 有効（`IsEnabled = true`）なシートのみをリストアップする。
            3. `GetDownloadUrl` でエクスポートURLを生成し、`FetchText` で順番にダウンロードを実行する。
            4. 仮想的なファイル名とダウンロードした生のテキスト（TSV文字列）をDictionaryに格納し、実際のパース処理（`ProcessImport`）へ渡す。
        - また、処理の冒頭で `EditorUtility.DisplayDialog` を表示し、誤操作によるデータの上書きを防ぐ設計にしている。
        
        <br>

    - **3-2. FetchText()メソッド （非同期通信処理）**
        - コード例
        
        ```csharp
        using UnityEngine.Networking;
        using System.Threading.Tasks;
        
        // 中略
        
        private static async Task<string> FetchText(string url)
        {
            using (var request = UnityWebRequest.Get(url))
            {
                var operation = request.SendWebRequest();
                while (!operation.isDone) await Task.Yield();
        
                if (request.result != UnityWebRequest.Result.Success)
                {
                    throw new System.Exception($"Request Error: {request.error} (Code: {request.responseCode})");
                }
                return request.downloadHandler.text;
            }
        }
        ```
        
        - **役割と仕組み**: 指定されたURLに対し、`UnityWebRequest.Get` を用いてテキストデータをダウンロードする処理。
        - **解説**: HTTP通信の待機に `async / await` を導入している。エディタ拡張の処理中に `while (!operation.isDone) await Task.Yield();` を挟むことで、ダウンロード中もUnityエディタがフリーズしない（バックグラウンドで処理が進む）設計になっている。
        

### 3. おわりに

- 厳密なことを言うと、OAuth等を利用してAPIで認証した方がセキュアではある。しかしまあ、個人開発においてそこまで厳密なものを追求しなくても良いように思う。今回は手軽さを優先した。
- 今回は「ダウンロード」する手順の解説のみだが、逆に「アップロード」も自動化が可能。 **GitHub Actionsを利用することで、コミットする度に指定したシートの更新を行うこともできる。** ただ、環境構築の手順が少し複雑になるため今回は割愛した。そのうちこの話も書くかもしれない。
