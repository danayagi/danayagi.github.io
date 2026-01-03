---
layout: post
title:  "Live2Dの表情差分設定をChatGPTとPythonで自動化する"
date:   2026-01-03
categories: Live2D RPA
key: 2026-01-03-Live2DFacialPartsRPA
---
### 1. はじめに（概要）

- **1-1. やったこと:**
    - Live2Dモデルにおける「表情差分」のデフォーマ構成作業を自動化した。メッシュの変形（モデリング）そのものではなく、定型的だが手間のかかる「デフォーマ階層の作成・設定」を自動で行うことができる。透明度設定やキーフォーム追加までは行うが、頂点編集などは行えない。
- **1-2. 仕組み**:
    - ①Live2Dエディタのスクリーンショットを撮影。
    - ②ChatGPT APIが画像を解析し、パーツパレット上の表情パーツを検出・分類。
    - ③Pythonが解析結果に基づき、マウス・キーボード操作 (RPA) を実行する。
- **1-3. 成果**:
    - 半年～1年ほど実際の案件で運用し、概ね実用レベルで稼働した。
- **1-4. 記事の趣旨**:
    - ツール配布ではなく、こういうアプローチでの自動化事例の紹介。

{: .caption-text }
▼動作例（GIF）。マウス操作もキー入力も全て自動で行っている。
![GIF_動作例.gif](/assets/img/20260103/GIF_動作例.gif)


### 2. 背景：なぜ自動化が必要だったか

- **2-1. 要件：CG集の映像化**
    - 「CG集の映像化」というタイプの企画に何件か関わり、実際の映像制作（Live2Dモデルの作成～MP4出力まで）を担当していた。
    その際、各シーン（一枚絵）＝各Live2Dモデルごとに「表情差分」の設定が必要となったのだが、1つの動画につき差分の総数はおおよそ **30～40パターン** に及び、単純作業ながら工数が多い箇所となっていた。

- **2-2. 定型化されたデータ構造**
    - それぞれの「表情差分」については、原画の段階でほとんどフォーマットが決まっていた。具体的には、
        1. **フォルダ構成**: 表情差分パターンごとに、1つのフォルダ内に必要なパーツがまとめられている。
        2. **目・口パーツ**: 「開いた状態」と「閉じた状態」のペアが存在する（※「開いた目」がない場合もある）。
        3. **その他のパーツ**: 眉や鼻など。変形はしないが、差分ごとに切り替える必要がある。


    {: .caption-text }
    ▼クリスタ上でのレイヤー構成（例）。こういう「表情差分」が各モデルごとに3～6パターンくらいある。<br>
    ![クリスタ上でのレイヤー構成](/assets/img/20260103/2-1.png){: .figure-image }

    ▼画像としてはこのような状態<br>
    ![画像としてはこのような状態](/assets/img/20260103/2-2.png){: .figure-image .small }


    - 原画が同じ形式であるため、Live2D上のデフォーマ構成も（それがどのようなシーン・表情の差分であれ）同じ形式になる。具体的には、以下のようなデフォーマ構成を毎回作成することになる。
        1. **開いた目・口**
            - 表示状態管理用の「スイッチャー」、変形用の「スケーラー」の2種類のデフォーマ。
        2. **閉じた目・口**
            - 「スイッチャー」のみ。
        3. **その他のパーツ**
            - 必要なし。
        4. **差分全体**
            - これらをまとめる親デフォーマと「スイッチャー」。

    {: .caption-text }
    ▼Live2Dエディタ上での最終的なデフォーマ構成<br>
    ![Live2Dエディタ上での最終的なデフォーマ構成](/assets/img/20260103/2-3.png){: .figure-image }


- **2-3. データの仕様と「表記揺れ」の壁**
    - 構造自体は定型化されていたため、差分フォルダに含まれる「表情パーツ」のリストさえあれば自動化自体は容易だった。しかし、その「リストを正確に生成する」ことが困難だった。
        - **問題①: APIの不在**
            - そもそもLive2DエディタにはAPIが存在せず（厳密にはあるらしいのだが、少なくともパーツ名称の取得やデフォーマ作成を行う手段はない）、 **外部からパーツ情報を取得することが難しい。**
        - **問題②：レイヤー名の「表記揺れ」と「文脈依存」**
            - 仮にパーツ情報を取得できたとしても、レイヤー名には「目」「開き目」「開いた目 コピー」といった**表記揺れ**が存在する。
            - また、「開いた目」を慣例的に単に「目」と命名しているケースなど、「フォルダ内の他パーツ（閉じ目など）の有無」から正体を推論しなければならないことも多い。そのため、正規表現などで**機械的に判別するのは困難**だった。

        - **問題③：正規化（手動リネーム）運用の煩わしさ**
            - 問題②については、誰が悪いとかそういう話ではなく、もし仮に「開いた目のレイヤーは"開いた目"として命名する」という規則を設けたとしても、そもそも（誰であれ）人間が目視で確認し、正規化のために手動リネームを行う運用は、ミスが起きやすく精神的負荷も高い（有り体に言うと面倒くさい）。
            - 例えば、クリスタのオートアクション等で事前にレイヤー名を設定することは可能だが、Live2D工程側でミスが発覚した場合、結局は手作業での修正が必要になってしまう。
    
    - **結論**: 
        - 表情差分の数だけ発生する「何回も繰り返す処理」であることを考慮すると、厳格なルールを前提とするよりも、**多少の表記揺れがあっても柔軟に吸収できる仕組み**が理想的だった。


### 3. 技術的アプローチ（解決策）

- **3-1. なぜLLM（ChatGPT）を採用したか**
    - スクリーンショットの情報を起点にすれば、「そのフォルダ内にある」パーツリストの作成が可能であることに気づいたため。
        - 「日本語」のOCR（画像からの文字認識）については、Pythonのライブラリ（pytesseract）では精度に問題があった。対して、**LLMは精度・費用ともに現実的なラインにある**。

        {: .caption-text }
        ▼ChatGPTで2回スクリーンショットを解析した費用は0.01ドル<br>
        ![ChatGPTでの解析費用](/assets/img/20260103/3-1.png){: .figure-image }
    - また、LLMであれば **「曖昧な入力」を分類できる** ため、前述した「表記揺れがあるデータ」にも対応が可能。
    - これら2つの機能が、単一のAPIで利用できる点が採用の決め手となった。

<br>
- **3-2. システム構成**
    1. **入力**: 画面のスクリーンショット。
    2. **脳（AI）**: 
        - **ChatGPT API**。画像を解析し、JSONデータを返す。
        - 出力されるJSONはパーツパレット上の並び順に対応しており、これを元に操作タスクリストを生成する。
    3. **手（Python）**: 
        - 返ってきたJSONに基づき、**PyAutoGUI**等を使用してGUI操作を実行する。
        - ※ワープデフォーマの作成やパラメータ作成については、既にモジュール化したものを使用している。

        <br>
        ▼デフォーマを作成するモジュールの例。色々やばい実装だが、こういう水準のコードでも動作自体はする。依存関係は割愛。

        ```python
        class DeformerManager:
            def __init__(self, screen_tool, image_locate_caches):
                self.screen_tool = screen_tool
                self.image_locate_caches = image_locate_caches
                self.deformer_icon_coords = None

            def create_warp_deformer(self, target_part_name):
                # 現在の作業ディレクトリを保存
                original_dir = os.getcwd()
                # スクリプトファイルのディレクトリに変更
                os.chdir(os.path.dirname(os.path.abspath(__file__)))

                # 座標が必要になるタイミングで取得
                warp_icon_coords = self.image_locate_caches.get_coordinate('warp_deformer_icon', 'images/warp_deformer_icon.png')
                # デフォーマ作成処理...
                pyautogui.click(warp_icon_coords)

                time.sleep(0.1)

                pyperclip.copy(target_part_name) 
                warp_deformer_window_center_coords = self.image_locate_caches.get_coordinate('warp_deformer_window', 'images/warp_deformer_window.png')
                self.new_deformer_setup(warp_deformer_window_center_coords)

                # 作業ディレクトリを元に戻す
                os.chdir(original_dir)

            # ワープデフォーマウィンドウを操作して、ワープデフォーマを作成する処理
            def new_deformer_setup(self, coords):
                # ウィンドウ内の相対座標を計算してクリック
                name_area = (coords[0], coords[1] - 115)
                pyautogui.click(name_area)
                time.sleep(0.01)

                # 「名前」欄を全選択する
                pyautogui.hotkey('ctrl', 'a') 
                time.sleep(0.01)

                pyautogui.press('delete')
                time.sleep(0.01)

                # コピペして文字入力を行う
                pyautogui.hotkey('ctrl', 'v') 
                time.sleep(0.01)

                # 「名前」を選択中であるという想定なので、タブを4回押して「OK」にカーソルを合わせる
                pyautogui.press('tab')  # Tabキーを押す
                time.sleep(0.01)
                pyautogui.press('tab')  # Tabキーを押す
                time.sleep(0.01)
                pyautogui.press('tab')  # Tabキーを押す
                time.sleep(0.01)
                pyautogui.press('tab')  # Tabキーを押す
                time.sleep(0.01)

                pyautogui.press('enter')  # Enterキーを押す
        ```



### 4. 実際の処理フロー

- **4-1. ①APIを叩いてスクリーンショットから「顔パーツリスト」のJSONを生成。配置されているパーツを5種類に分類する**。

    ▼プロンプトの実装例（python）

    ```python
    prompt = """
        以下に添付するのはLive2DのUI上に表示されているパーツ群です。
        このとき、左端の目のアイコンが設定されている、表示中の表情グループについてのみ、以下の文字列（＝パーツ）が表示されている順番をリストアップしてください。絶対に数値でソートしないでください。画面に表示されている順番のまま、上から順番にリストアップしてください。絶対にソートしないでください。迷惑です。
        表示されているパーツ名称：（下記の5分類のうちどれに該当するか、1～5の数値）、というフォーマットで記述してください。絶対に数値でソートしないでください。また、半角の数字（1,2,3...というように）で出力してください。他の形式にはしないこと。
        解析は手動で行い、結果はjson形式で出力してください。json形式のみで、どのような場合であっても説明や解説などは一切不要です。
        また、コピー元のパーツがない場合でも、「コピー」のような名前になっているパーツがあるので、注意してください。そのようなパーツがあっても元のパーツが存在するとは限りません。どのような場合であっても、名前を取得する際は画面情報から読み取ってください。
        パーツ名に英数字が含まれている場合は半角にしてください。
        ①～⑤の数値に該当するパーツが存在しない場合もまれにあるので、あくまでも画像データに準拠した解析データを作成してください。絶対に数値でソートせず、そのままの順番でjsonにしてください。

        例：
        [
        {"displayed_expression_group": "（「表示中の表情グループ」のフォルダ名）"},
        {"part": "右目_o", "category": 1}, ...
        ]

        ①開いた状態（ = 'o'）の目、または特に指定のない「目」のパーツ
        ②閉じた状態（ = 'c') の目
        ③開いた状態（ = 'o'）の口、または特に指定のない「口」のパーツ
        ④閉じた状態（ = 'c') の口
        ⑤その他の顔パーツ（全てリストアップすること。）
        """
        
        headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {api_key}"
        }
        
        payload = {
            "model": "gpt-4o-2024-08-06",
            "messages": [
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "text",
                            "text": prompt
                        },
                        {
                            "type": "image_url",
                            "image_url": {
                                "url": f"data:image/jpeg;base64,{base64_image}"
                            }
                        }
                    ]
                }
            ],
            "max_tokens": 500
        }
    ```

    <br>
    {: .caption-text }
    ▼ChatGPT（API）に渡すスクリーンショットの例。これもpythonで撮影と送信を行う。<br>
    ![スクリーンショット](/assets/img/20260103/4-1_ss.png){: .figure-image }

    <br>
    ▼生成されるjsonデータの例

    ```json
    [
        {
            "displayed_expression_group": "表情標準"
        },
        {
            "part": "目のコピー9",
            "category": 1
        },
        {
            "part": "閉じた目",
            "category": 2
        },
        {
            "part": "口",
            "category": 3
        },
        {
            "part": "閉じ口",
            "category": 4
        },
        {
            "part": "眉",
            "category": 5
        }
    ]
    ```

    ※メモ：「開いた目・口」と「閉じた目・口」で処理フローに差はないので、「開いたパーツ」と「閉じたパーツ」という粒度で分類してもよかったかもしれない

<br>

- **4-2. ②Pythonで「パーツリスト」と「タスクリスト」のJSONを生成する**
    - APIから返ってきた分類データを元に、具体的な作業タスクを作成する。「目」や「口」ならデフォーマ作成タスクを追加し、「その他」なら親デフォーマ作成用のマーカーだけを打つ、といった振り分けを行う。

    <br>
    ▼実装例（タスク生成ロジック）

    **※実装コードについて**
    <br>
    以下は、処理の核心部分を抜粋したコードです。
    実際には画像の座標特定や依存関係の解決を行う自作クラス（`MarkerListClass`, `ImageLocateCaches` 等）を呼び出しているため、このままコピペしても動作はしませんが、「AIの出力をどうプログラムで受け止めているか」のロジック参照用として掲載します。

    ```python
    def MakePartList(part_count,expression_group_from_json, parts_from_json):
        partList = MarkerListClass(part_count, 0)
        taskList = TaskList.TaskListClass(Live2DTaskScenarioNames)

        taskNumber = 1

        # フォルダ作成は初手に置く
        taskList.add_task(taskNumber, Live2DTaskScenarioNames.TASK_CREATE_PARAMETER_FOLDER, text=expression_group_from_json)

        # "part"の内部構造に応じてループ
        for index, part in enumerate(parts_from_json):
            part_name = part['part']
            category = part['category']

            # 現在ループしているアイテムから名前を取得し、パーツリストを設定
            partList.set_value(index, part_name)

            # カテゴリーが特定の番号であるかを調べる
            # 「開いた」状態のパーツである場合、「オープナー」マーカーを設置する
            if category == 1 or category == 3:
                partList.add_marker(index, 'OPENER')

                taskNumber += 1
                # タスク編集
                add_scaler_tasks(taskNumber, taskList, part_name)
                add_switcher_tasks(taskNumber, taskList, part_name)

            # 「閉じた」状態のパーツである場合、「スイッチャー」マーカーを設置する
            elif category == 2 or category == 4:
                partList.add_marker(index, 'SWITCHER')

                taskNumber += 1
                # タスク編集
                add_switcher_tasks(taskNumber, taskList, part_name)

            # 「それ以外の顔パーツ」である場合、「ペアレント」マーカーを設置する
            elif category == 5:
                partList.add_marker(index, 'PARENT')

            # 上記以外の場合は例外処理を行う
            else:
                raise ValueError(f"Unexpected category {category} for part: {part_name}")
            
        taskNumber += 1

        # 「whole」デフォーマの作成（タスク）
        wholeDeformerName = expression_group_from_json + "_whole"
        taskList.add_task(taskNumber, Live2DTaskScenarioNames.TASK_CREATE_WHOLE_DEFORMER, text=wholeDeformerName)

        # 全体のスイッチャーの作成（タスク）
        add_switcher_tasks(taskNumber, taskList, wholeDeformerName)
            
        return partList, taskList
    ```

<br>

- **4-3. ③Python（PyAutoGUI）でタスクを実行する**
    - 生成されたタスクリストを順に実行し、実際にLive2Dエディタを操作する。
    パーツリストの位置情報は動的に変わる（デフォーマを作ると行が増える）ため、`partList` オブジェクト側でインデックスを管理しながらクリック位置を計算している。

    <br>
    ▼実装例（実行エンジン）

    ```python
    # タスクリストの情報を元にタスクを行い、パーツ追加（デフォーマ追加）した際はパーツリストを更新する
    def doTask(partList, taskList):
        global anchorPointCoords

        image_locate_caches = ImageLocateCaches()
        deformer_manager = DeformerManager(screen_tools, image_locate_caches)
        parameter_manager = ParameterManager(screen_tools, image_locate_caches)

        currentTaskNumber = 0
        for task in taskList:
            # 緊急停止用
            if keyboard.is_pressed("esc"):
                print("処理が中断されました")
                break

            # タスク番号の検出。現在処理中のタスク番号に変化があれば、対象パーツのクリック等を行う。
            if task.task_id != currentTaskNumber:
                currentTaskNumber = task.task_id
                targetPartIndex = partList.search_last_occurrence(task.text)
                if targetPartIndex != -1:
                    # 対象パーツのクリック処理
                    currentTargetCoords = adjust_y_coordinate(anchorPointCoords, targetPartIndex+1, 'down')
                    pyautogui.click(currentTargetCoords)
            

            # 実際のタスクを行う。「完了」以外であれば処理を行う
            try:
                if task.status.value == TaskList.TaskStatus.COMPLETED:
                    continue
                
                match task.name:
                    # 「パラメータフォルダ」の作成
                    case Live2DTaskScenarioNames.TASK_CREATE_PARAMETER_FOLDER:
                        parameter_manager.createFolder()

                    # 「スイッチャーデフォーマ」の作成
                    case Live2DTaskScenarioNames.TASK_CREATE_SWITCHER_DEFORMER:
                        deformer_manager.create_warp_deformer(task.text)
                        targetPartIndexTmp = partList.search_first_occurrence(task.text)
                        if targetPartIndexTmp != -1:
                            partList.insert_at(targetPartIndexTmp, task.text)
                            partList.add_marker(targetPartIndexTmp, 'PARENT')
                        else:
                            raise ValueError(f"'{task.text}' was not found in partList.")

                    # 「スイッチャーパラメータ」の作成
                    case Live2DTaskScenarioNames.TASK_CREATE_SWITCHER_PARAMETER:
                        parameter_manager.createParameter(task.text)

                    # 「スイッチャーパラメータ」の編集
                    case Live2DTaskScenarioNames.TASK_MODIFY_SWITCHER_PARAMETER:
                        parameter_manager.setSwitcherValues()

                    # 「スケーラーデフォーマ」の作成
                    case Live2DTaskScenarioNames.TASK_CREATE_SCALER_DEFORMER:
                        deformer_manager.create_warp_deformer(task.text)
                        targetPartIndexTmp = partList.search_first_occurrence(task.text)
                        if targetPartIndexTmp != -1:
                            partList.insert_at(targetPartIndexTmp, task.text)
                        else:
                            raise ValueError(f"'{task.text}' was not found in partList.")

                    # 「スケーラーパラメータ」の作成
                    case Live2DTaskScenarioNames.TASK_CREATE_SCALER_PARAMETER:
                        parameter_manager.createScalerParameter(task.text)

                    # 「スケーラーパラメータ」の編集
                    case Live2DTaskScenarioNames.TASK_MODIFY_SCALER_PARAMETER:
                        parameter_manager.modifyScalerParameter()

                    # 「全体デフォーマ」の作成
                    case Live2DTaskScenarioNames.TASK_CREATE_WHOLE_DEFORMER:
                        # まずパーツリストに設定されている「PARENT」マーカーの箇所を複数選択する
                        indices = partList.get_indices_with_marker('PARENT')
                        if len(indices) >= 1:
                            for index in indices:
                                # 対象パーツのクリック処理
                                currentTargetCoords = adjust_y_coordinate(anchorPointCoords, index+1, 'down')
                                pyautogui.click(currentTargetCoords)

                                time.sleep(0.1)
                                pyautogui.keyDown('ctrl')
                                time.sleep(0.1)
                            pyautogui.keyUp('ctrl')
                            # この時点で「PARENT」マーカーのあるパーツは全てクリックされている（想定）
                            deformer_manager.create_warp_deformer(task.text)
                            # 全体デフォーマの位置はどうでもいいはずなのでプログラム上の位置は先頭固定でいいはず
                            partList.insert_at(0, task.text)
                        else:
                            raise ValueError(f"No parts with the 'PARENT' marker were found in partList. Task '{task.text}' cannot be completed.")

                    case _:
                        raise Exception("task hanbetu fukanou")
            except Exception as e:
                print(f"An error occurred: {e}")
                taskList.update_task_status(task.name, TaskList.TaskStatus.FAILED, task.text)
                raise e

            # ここからが正常系の処理
            taskList.update_task_status(task.name, TaskList.TaskStatus.COMPLETED, task.text)
            time.sleep(0.5)

        print('完了')
    ```

<br>

- **4-4. 実行結果**

{: .caption-text }
▼動作例（GIF）。再掲。
![GIF_動作例.gif](/assets/img/20260103/GIF_動作例.gif)

{: .caption-text }
▼処理完了後。表示切り替え用のパラメータは正常に設定されている
![GIF_動作例_4-1.gif](/assets/img/20260103/4-1_GIF.gif)


### 5. 開発での「つまづき」と知見

- **5-1. プロンプトエンジニアリングの壁**
    - **課題**: 
        - AI（ChatGPT）が、画面上の「見た目の並び順」ではなく、カテゴリーの数値順などで勝手に **リストをソートして出力してしまう** 現象が多発した。
    - **影響**:
        - スクリプトは上から順番に処理を行う前提で作られているため、「見た目の表示順」と「スクリプトの実行順」がズレると、違うパーツを操作してしまい正常に動作しなくなる。体感では、不具合の原因としてこれが一番多かった。
    - **詳細**:
        - 特に、今回使用した `gpt-4o-2024-08-06` は、データを論理的に整理整頓しようとするバイアスが強いのか、「カテゴリー」の数値で昇順ソートしたがる傾向が見られた。

        {: .caption-text }
        ▼Live2Dエディタでは「眉」が一番 **上** にある。<br>
        ![5-1_ソート例1.png](/assets/img/20260103/5-1.png)

        ▼勝手にソートしてしまう例。API経由で解析させたJSONデータでは、「眉」が一番 **下** にある（カテゴリで昇順ソートしてしまっている？）。<br>
        ![5-2_ソート例2.png](/assets/img/20260103/5-2.png)

    <br>
        
    - **知見**:
        - プロンプトで強く制約することで正常に動くこともあったが、それでも時々ソートされて出力されることがあった。
        - 最も安定させるには、**「画面上の配置自体を、最初から昇順ソートになるように並べ替えておく（＝AIがソートしても順序が変わらない状態にする）」** という、人間側での運用回避が有効であるように思う。

        {: .caption-text }
        ▼今回の例だと、Live2Dエディタでも「眉」を一番下に配置すると安定する。RPA処理後に元の位置に戻す手間もほぼない。<br>
        ![5-3_対策.png](/assets/img/20260103/5-3.png)

<br>

- **5-2. 画像認識の壁**
    - **課題**:
        - スクリーンショットを「原寸サイズ（1.0倍）」のままAIに渡すと、解析エラーや認識漏れが発生することが多かった。
    - **解決策**:
        - **「画像を1.5倍に拡大してからAPIに投げる」** ことで、途端に精度が安定した。
        - 従来のOCRライブラリ（pytesseract等）では、読み取れなかった文字が関係のない漢字に置き換わるといった誤認識が頻発したが、LLM（1.5倍拡大済み）では **文字認識そのものがボトルネックになることはほぼ無かった**。

<br>

- **5-3. RPA（GUI自動化）の壁**
    - **IMEの問題**:
        - 日本語入力（IME）が「全角モード」になっていると、特に**数値入力**が正常に処理されない。Live2Dエディタ上でのパラメータ設定などで数値が設定できず、フロー全体が異常終了する原因となる。（多分python側で簡単に対策することは出来る）
    - **フォーカスとタイミング**:
        - 非アクティブウィンドウへの操作や、ラグ等でポップアップが出るのが遅れた場合の例外処理。これらはAPI操作にはない、GUI自動化特有の不安定要素であり、`time.sleep` の調整や画像認識による待機処理が必要になる。


### 6. 課題と感想

- **6-1. 画面依存の脆さ**
    - **UIの見切れ**: 
        - 「差分」フォルダ全体がパーツパレットの下の方にある場合、デフォーマを追加していく度にそれよりも「下」にあるパーツがさらに下に移動してしまうため、処理の途中で見切れるパーツが発生してしまう。**画面上でスクロールしないと見えない（見切れている）ものは、物理的にクリックができないため操作不能**になる。

            {: .caption-text }
            ▼このあたりで処理を開始すると確実に見切れる（特に「口」「閉じ口」などがウィンドウの外に出てしまうので、対応するデフォーマを作成できない）。画面上に「存在しない」パーツは選択不可能。<br>
            ![6-1.png](/assets/img/20260103/6-1.png)

    <br>

    - **レイアウト変更への弱さ**: 
        - Live2Dエディタのバージョンアップでアイコンやウィンドウのデザインが変わると、Python側の画像認識処理がマッチしなくなり動作しなくなる。実際、デフォーマウィンドウの識別などがバージョンアップで動かなくなることがあった。

    <br>

    - **誤検出**: 
        - 時折、指定した「差分」フォルダの外にあるパーツまでAIが拾ってしまうことがあった。
        
            {: .caption-text }
            ▼処理対象の「表情差分」のすぐ下に、全ての「表情差分」で共通のパーツ（「赤面2」「赤面」）が配置されてしまっている例。<br>
            ![6-2.png](/assets/img/20260103/6-2.png)
            
            ▼生成されたJSON。「赤面2」と「赤面」のパーツが不要。
            ```json
            [
                {"displayed_expression_group": "表情_02"},
                {"part": "眉", "category": 5},
                {"part": "目", "category": 1},
                {"part": "閉じ目", "category": 2},
                {"part": "口", "category": 3},
                {"part": "閉じ口", "category": 4},
                {"part": "赤面2", "category": 5},
                {"part": "赤面", "category": 5}
            ]
            ```
    
    <br>
    
- **6-2. 実行速度とコスト、およびリスク**
    - **速度**: 
        - APIを叩いてからJSONが返ってくるまで大体**約5秒程度**だった。手作業の手間を考えれば、待ち時間は全く気にならないレベルだった。
    - **コスト**: 
        - 今回の規模感であれば、API利用料も微々たるもので無視できる範囲。
    - **コンテンツリスク（未検証）**:
        - 解析にスクリーンショットを用いているため、もし画像内にOpenAIの定める「不適切な表現・単語（NSFW等）」が含まれる場合、API側でリジェクトされる可能性がある（今回は発生しなかったが、可能性としては考慮すべき）。

<br>

- **6-3. 結論**
    - まれに想定とは異なる動作をする場合もあったが、今回のような **「誰がやっても同じ結果になる（なるべき）」定型作業** は自動化しやすく、その恩恵も大きい。処理フローの一部としてAIにOCR処理をさせるという手法は、既に十分実用的な水準にあると感じた。


### 7. おわりに

- APIがないツールでも、現行水準のAIなら**人間のように画面の情報を理解できる**ため、それを足がかりに自動化できる場合がある。