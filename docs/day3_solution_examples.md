# Day 3 オリジナル機能 解答例

> 各テーマの「追加する機能（例）」の実装例と期待される出力を示します。  
> 自分で先に実装してから参照することを推奨します。

---

## 目次

- [テーマA：住民窓口](#テーマa-住民窓口)
- [テーマB：土木・インフラ](#テーマb-土木インフラ)
- [テーマC：福祉・子育て](#テーマc-福祉子育て)
- [テーマD：教育](#テーマd-教育)

---

## テーマA：住民窓口

### 機能1：必要書類チェックリスト生成

**実装のポイント：** 手続き名からRAGで関連情報を取得し、LLMにチェックボックス形式で出力させる。

```python
def generate_checklist(self, procedure: str) -> str:
    relevant = self.search(procedure, n_results=3)
    context = '\n\n'.join(relevant)
    prompt = f"""
    以下の参考情報をもとに、「{procedure}」の必要書類チェックリストを作成してください。

    出力形式：
    ## {procedure} 必要書類チェックリスト

    ### 必ず持参するもの
    - [ ] 書類名（補足説明）

    ### 場合によって必要なもの
    - [ ] 書類名（どんな場合に必要か）

    ### 手数料・費用
    - 金額と支払い方法

    ### 窓口・受付時間
    - 場所と時間

    参考情報：{context}
    """
    return model.generate_content(prompt).text
```

**期待される出力例：**

```
## 転入届 必要書類チェックリスト

### 必ず持参するもの
- [ ] 転出証明書（前住所の市区町村で発行してもらう）
- [ ] 本人確認書類（運転免許証・パスポート・マイナンバーカード等）

### 場合によって必要なもの
- [ ] マイナンバーカード（お持ちの場合：住所変更手続きに必要）
- [ ] 国民健康保険証（加入中の場合：切り替え手続き）
- [ ] 印鑑（認印でも可）

### 手数料・費用
- 転入届自体は無料
- 住民票の写しが必要な場合：300円（窓口）/ 200円（コンビニ）

### 窓口・受付時間
- 市役所1階 市民課
- 平日 8:30〜17:15
- 第2・第4土曜 9:00〜12:00
```

---

### 機能2（発展）：混雑時間帯を考慮した来庁アドバイス

**実装のポイント：** 訪問目的・曜日・時間帯を入力し、最適な来庁タイミングを提案する。

```python
def suggest_visit_time(self, procedure: str, preferred_day: str = '') -> str:
    """来庁タイミングのアドバイスを生成（オリジナル機能）"""
    relevant = self.search('混雑 窓口', n_results=2)
    context = '\n\n'.join(relevant)
    day_hint = f'希望曜日：{preferred_day}' if preferred_day else '曜日の希望なし'

    prompt = f"""
    「{procedure}」の手続きのために市役所に来庁する際の、最適な時間帯をアドバイスしてください。
    {day_hint}

    以下の情報を含めて回答してください：
    1. おすすめの曜日・時間帯（理由付き）
    2. 避けるべき曜日・時間帯（理由付き）
    3. 待ち時間を短縮するコツ（事前予約・書類準備など）

    参考情報：{context}
    """
    return model.generate_content(prompt).text
```

**期待される出力例：**

```
「転入届」の手続きのための来庁アドバイスです。

✅ おすすめの曜日・時間帯
- 火〜木曜日の 10:00〜11:30 または 14:00〜16:00
  → 月曜・週末明けを避けることで待ち時間が大幅に短縮されます。
- 月の中旬（5〜20日頃）も比較的空いています。

⚠️ 避けるべき曜日・時間帯
- 月曜日の午前中（週の始まりで集中しやすい）
- 月末・月初（住民票や証明書の需要が高まる）
- 3月〜4月の引越しシーズン（転入届が最も集中する時期）

💡 待ち時間を短縮するコツ
1. 転出証明書を事前に取得しておく
2. 本人確認書類を複数持参する（有効期限の確認も）
3. マイナンバーカードをお持ちの場合、転出届はマイナポータルからオンライン完結が可能
4. 市民課（000-0000-0000）に電話して、当日の混雑状況を事前確認する手も有効です
```

---

### Gradio UI 統合例（テーマA完成版）

```python
import gradio as gr

bot_a = WindowBotA()

def chat_fn(message, history):
    return bot_a.answer(message)

def checklist_fn(procedure):
    if not procedure.strip():
        return '手続き名を入力してください（例：転入届、印鑑登録）'
    return bot_a.generate_checklist(procedure)

def visit_advice_fn(procedure, preferred_day):
    if not procedure.strip():
        return '手続き名を入力してください'
    return bot_a.suggest_visit_time(procedure, preferred_day)

with gr.Blocks(title='🏠 住民窓口AIアシスタント', theme=gr.themes.Soft()) as demo:
    gr.Markdown('# 🏠 住民窓口AIアシスタント')
    gr.Markdown('転入届・印鑑登録・マイナンバーカードなどの手続きをご案内します。')

    with gr.Tabs():
        with gr.Tab('💬 チャット'):
            gr.ChatInterface(fn=chat_fn, examples=[
                '転入届に必要なものを教えてください',
                '印鑑登録はどこでできますか？',
                'マイナンバーカードを作りたいです'
            ])

        with gr.Tab('📋 必要書類チェックリスト'):
            gr.Markdown('手続き名を入力すると、必要書類のチェックリストを自動生成します。')
            procedure_input = gr.Textbox(label='手続き名', placeholder='例：転入届、印鑑登録')
            checklist_output = gr.Markdown()
            gr.Button('チェックリストを生成').click(
                checklist_fn, inputs=procedure_input, outputs=checklist_output
            )

        with gr.Tab('🕐 来庁アドバイス'):
            gr.Markdown('手続きと希望曜日を入力すると、最適な来庁タイミングをアドバイスします。')
            proc_input2 = gr.Textbox(label='手続き名', placeholder='例：転入届')
            day_input = gr.Dropdown(
                label='希望曜日（任意）',
                choices=['', '月曜', '火曜', '水曜', '木曜', '金曜', '土曜'],
                value=''
            )
            advice_output = gr.Markdown()
            gr.Button('アドバイスを表示').click(
                visit_advice_fn, inputs=[proc_input2, day_input], outputs=advice_output
            )

demo.launch(share=True)
```

---

## テーマB：土木・インフラ

### 機能1：緊急度自動分類

**実装のポイント：** LLMにJSON形式で返答させ、`re` でJSONを抽出して構造化データとして扱う。  
JSON解析の失敗に備えたフォールバック処理が重要。

```python
import json, re

def classify_urgency(self, report: str) -> dict:
    prompt = f"""
    以下の道路損傷報告の緊急度を判定してください。

    緊急度の基準：
    - 高：人身事故・通行不能のリスクあり（深い穴・ガードレール破損・大規模陥没など）
    - 中：放置すると危険になる可能性あり（白線消え・軽度の舗装劣化・側溝の蓋ずれ）
    - 低：軽微な損傷（小さなひび割れ・軽微な段差・側溝の汚れ）

    報告内容：{report}

    必ずJSON形式のみで返答してください（説明文は不要）：
    {{"urgency": "高 or 中 or 低", "reason": "判定理由を1文で", "action": "推奨する対応を1文で"}}
    """
    response = model.generate_content(prompt).text
    # JSON部分だけを抽出（LLMが余計な文章を付けることがあるため）
    match = re.search(r'\{{[^}}]+\}}', response, re.DOTALL)
    if match:
        try:
            return json.loads(match.group())
        except json.JSONDecodeError:
            pass
    # パース失敗時のフォールバック
    return {
        'urgency': '中',
        'reason': '自動判定できませんでした',
        'action': '建設課道路維持係に詳細をお問い合わせください'
    }
```

**期待される出力例：**

```python
# 入力例1：緊急度「高」
report1 = '国道沿いのガードレールが車の衝突でひしゃげています。今にも倒れそうです。'
# → {'urgency': '高', 'reason': 'ガードレールが倒壊寸前で二次事故のリスクが高い', 'action': '24時間以内に現地確認・緊急補修または通行規制を実施'}

# 入力例2：緊急度「中」
report2 = '交差点付近の横断歩道の白線がほとんど消えています。歩行者が危険です。'
# → {'urgency': '中', 'reason': '白線の消えは視認性低下につながり放置すると危険', 'action': '1週間以内に区画線の引き直しを実施'}

# 入力例3：緊急度「低」
report3 = '歩道のコンクリートに細いひびが入っています。段差はありません。'
# → {'urgency': '低', 'reason': '軽微なひび割れで即時の危険はない', 'action': '通常の維持管理サイクル内で補修を実施'}
```

---

### 機能2：報告票自動生成

**実装のポイント：** 緊急度分類の結果を組み合わせて、構造化された報告票をMarkdownで出力する。

```python
def generate_report(self, location: str, damage_type: str, description: str) -> str:
    urgency_result = self.classify_urgency(description)
    urgency_label = self.URGENCY_LEVELS.get(urgency_result['urgency'], '🟡 優先対応（1週間以内）')
    now = datetime.now().strftime('%Y年%m月%d日 %H:%M')
    report_id = f"RD-{datetime.now().strftime('%Y%m%d%H%M%S')}"

    return f"""
## 道路損傷報告票

| 項目 | 内容 |
|------|------|
| 報告番号 | {report_id} |
| 受付日時 | {now} |
| 場所 | {location} |
| 損傷種別 | {damage_type} |
| **緊急度** | **{urgency_label}** |
| 判定理由 | {urgency_result['reason']} |
| 推奨対応 | {urgency_result['action']} |

### 詳細説明
{description}

### 対応担当
建設課道路維持係（TEL: 000-0000-0001）  
夜間・休日：市役所代表（TEL: 000-0000-0000）

---
*この報告票は自動生成されました。内容を確認のうえ担当者へ引き継いでください。*
"""
```

**期待される出力例：**

```
## 道路損傷報告票

| 項目 | 内容 |
|------|------|
| 報告番号 | RD-20260426143022 |
| 受付日時 | 2026年04月26日 14:30 |
| 場所 | ○○市△△町1-2-3 ○○交差点付近 |
| 損傷種別 | ポットホール（路面の穴） |
| **緊急度** | **🔴 緊急対応（24時間以内）** |
| 判定理由 | 深さ15cmの穴は自転車・バイクの転倒リスクが高い |
| 推奨対応 | 即日に応急補修（砂利・アスファルト充填）を実施し、必要に応じて通行規制を検討 |

### 詳細説明
深さ約15cm、幅約40cmの穴があります。自転車が転倒しそうで危険です。

### 対応担当
建設課道路維持係（TEL: 000-0000-0001）  
夜間・休日：市役所代表（TEL: 000-0000-0000）
```

---

### Gradio UI 統合例（テーマB完成版）

```python
bot_b = RoadDamageBot()

def chat_fn(message, history):
    return bot_b.answer(message)

def report_fn(location, damage_type, description):
    if not all([location.strip(), damage_type.strip(), description.strip()]):
        return '⚠️ 場所・損傷種別・詳細説明をすべて入力してください。'
    return bot_b.generate_report(location, damage_type, description)

with gr.Blocks(title='🛣️ 道路損傷報告AI', theme=gr.themes.Soft()) as demo:
    gr.Markdown('# 🛣️ 道路損傷報告AIアシスタント')

    with gr.Tabs():
        with gr.Tab('💬 チャット相談'):
            gr.ChatInterface(fn=chat_fn, examples=[
                '道路に大きな穴があります',
                'ガードレールが壊れています',
                '道路工事の許可はどこに申請しますか？'
            ])

        with gr.Tab('📋 報告票を作成'):
            gr.Markdown('情報を入力すると、緊急度を自動判定して報告票を生成します。')
            with gr.Row():
                location_in = gr.Textbox(label='場所（住所・目標物）', placeholder='例：○○市△△町1-2-3 ○○交差点付近')
                damage_type_in = gr.Dropdown(
                    label='損傷種別',
                    choices=['ポットホール（路面の穴）', 'ガードレール破損', '白線消え', '陥没', '側溝蓋ずれ', 'その他']
                )
            description_in = gr.Textbox(label='詳細説明', lines=4, placeholder='損傷の大きさ・状況・危険度などを具体的に記入してください')
            report_out = gr.Markdown()
            gr.Button('報告票を生成', variant='primary').click(
                report_fn,
                inputs=[location_in, damage_type_in, description_in],
                outputs=report_out
            )

demo.launch(share=True)
```

---

## テーマC：福祉・子育て

### 機能1：年齢別サービス一覧表示

**実装のポイント：** 月齢を整数で受け取り、年齢区分（0〜2歳・3歳〜就学前・小学生・中学生）を  
プログラム側で判定してからLLMに渡すと、出力が安定する。

```python
def get_services_by_age(self, age_months: int) -> str:
    if age_months < 12:
        age_str = f'{age_months}ヶ月'
        age_category = '0〜2歳（乳幼児）'
    elif age_months < 36:
        age_str = f'{age_months // 12}歳{age_months % 12}ヶ月'
        age_category = '0〜2歳（乳幼児）'
    elif age_months < 72:
        age_str = f'{age_months // 12}歳'
        age_category = '3歳〜就学前'
    elif age_months < 144:
        age_str = f'{age_months // 12}歳（小学生）'
        age_category = '小学生（6〜11歳）'
    else:
        age_str = f'{age_months // 12}歳（中学生）'
        age_category = '中学生（12〜14歳）'

    prompt = f"""
    お子さんの年齢：{age_str}（区分：{age_category}）

    以下の参考情報をもとに、この年齢のお子さんが利用できる市のサービスをすべて一覧にしてください。

    出力形式（Markdownの表）：
    | サービス名 | 内容 | 申請・利用窓口 | 備考（費用・期限など） |
    |-----------|------|--------------|----------------------|

    表の下に「今すぐ申請が必要なもの」があれば強調してください。

    参考情報：{THEME_C_FAQ}
    """
    return model.generate_content(prompt).text
```

**期待される出力例（8ヶ月の場合）：**

```
お子さん（8ヶ月）が利用できる市のサービス一覧

| サービス名 | 内容 | 申請・利用窓口 | 備考 |
|-----------|------|--------------|------|
| 児童手当 | 月15,000円を支給 | こども課（市役所3階） | 出生後15日以内に申請要 |
| 一時保育 | 通院・リフレッシュ等で一時的に保育 | 各保育所・子育て支援センター | 要予約／1時間200円〜 |
| 子育て支援センター | 自由遊び・育児相談・親子イベント | 子育て支援センター | 無料・予約不要 |
| 乳幼児健康診断 | 発育・発達の確認（市が無料実施） | 保健センター | 個別に案内が届きます |

⚠️ 今すぐ対応が必要なもの
**児童手当**：出生日の翌日から **15日以内** に申請しないと遡及支給が受けられません。  
まだお済みでない場合はこども課（000-0000-0003）へお急ぎください。
```

---

### 機能2（発展）：申請期限リマインダーメッセージ生成

**実装のポイント：** 子どもの誕生日・出生日などのイベントを受け取り、申請期限と必要手続きを計算して案内する。

```python
from datetime import date, timedelta

def generate_deadline_reminder(self, event: str, event_date: date) -> str:
    """申請期限リマインダーメッセージを生成（オリジナル機能）"""
    today = date.today()
    days_since_event = (today - event_date).days
    deadline_map = {
        '出生': 15,
        '転入': 15,
        '退職': 14,
    }
    deadline_days = next((v for k, v in deadline_map.items() if k in event), 14)
    deadline_date = event_date + timedelta(days=deadline_days)
    days_remaining = (deadline_date - today).days

    if days_remaining < 0:
        urgency_msg = f'⚠️ 申請期限（{deadline_date}）をすでに {abs(days_remaining)} 日過ぎています。至急窓口にご相談ください。'
    elif days_remaining <= 3:
        urgency_msg = f'🔴 申請期限まであと {days_remaining} 日です！今すぐ手続きを！'
    elif days_remaining <= 7:
        urgency_msg = f'🟡 申請期限まであと {days_remaining} 日です。今週中に手続きを。'
    else:
        urgency_msg = f'🟢 申請期限まであと {days_remaining} 日あります（期限：{deadline_date}）。'

    prompt = f"""
    「{event}」（{event_date}）があったご家庭へのリマインダーメッセージを作成してください。

    申請期限情報：{urgency_msg}

    以下の参考情報をもとに、必要な手続きと申請先を箇条書きで案内してください。
    温かみのある丁寧な文体で書いてください。

    参考情報：{THEME_C_FAQ}
    """
    result = model.generate_content(prompt).text
    return f'{urgency_msg}\n\n{result}'
```

**期待される出力例（出生から10日後に呼び出した場合）：**

```
🔴 申請期限まであと 5 日です！今すぐ手続きを！

おめでとうございます！お子さまのご誕生、心よりお祝い申し上げます。

お子さまが生まれてから15日以内に、以下の手続きをお忘れなくお済みください。

📋 必要な手続き
• 【児童手当の申請】こども課（市役所3階）
  → 支給額：月15,000円（0〜2歳）
  → 必要書類：認定請求書、健康保険証、通帳、マイナンバー書類

• 【子どもの健康保険加入】保険年金課（市役所2階）または勤務先の手続き

💡 まとめて手続きできます
市役所では児童手当と健康保険の手続きを同日に行うことができます。
不明な点はこども課（000-0000-0003）までお気軽にお問い合わせください。
```

---

### Gradio UI 統合例（テーマC完成版）

```python
bot_c = ChildcareBot()

def chat_fn(message, history):
    return bot_c.answer(message)

def age_service_fn(age_years, age_months_extra):
    total_months = age_years * 12 + age_months_extra
    return bot_c.get_services_by_age(total_months)

def reminder_fn(event, event_date_str):
    if not event_date_str:
        return '日付を入力してください'
    event_date = date.fromisoformat(event_date_str)
    return bot_c.generate_deadline_reminder(event, event_date)

with gr.Blocks(title='👨‍👩‍👧 子育て支援AIアシスタント', theme=gr.themes.Soft()) as demo:
    gr.Markdown('# 👨‍👩‍👧 子育て支援AIアシスタント')

    with gr.Tabs():
        with gr.Tab('💬 チャット相談'):
            gr.ChatInterface(fn=chat_fn, examples=[
                '保育所の申込みはいつですか？',
                '児童手当の金額を教えてください',
                '一時保育を利用したいです'
            ])

        with gr.Tab('📅 年齢別サービス一覧'):
            gr.Markdown('お子さんの年齢を入力すると、利用できるサービスを一覧で表示します。')
            with gr.Row():
                age_y = gr.Slider(0, 15, value=0, step=1, label='年齢（歳）')
                age_m = gr.Slider(0, 11, value=8, step=1, label='月齢（ヶ月）')
            service_out = gr.Markdown()
            gr.Button('サービスを調べる').click(
                age_service_fn, inputs=[age_y, age_m], outputs=service_out
            )

        with gr.Tab('⏰ 申請期限リマインダー'):
            gr.Markdown('イベントと日付を入力すると、申請期限と必要手続きをお知らせします。')
            event_in = gr.Dropdown(
                label='イベント',
                choices=['出生', '転入', '保育所入所決定']
            )
            date_in = gr.Textbox(label='イベント日（YYYY-MM-DD）', placeholder='例：2026-04-20')
            reminder_out = gr.Markdown()
            gr.Button('リマインダーを生成').click(
                reminder_fn, inputs=[event_in, date_in], outputs=reminder_out
            )

demo.launch(share=True)
```

---

## テーマD：教育

### 機能1：転校手続きフロー生成

**実装のポイント：** 状況（市内転居・他市転入・他市転出）を分岐させて、ステップと担当窓口を構造的に出力させる。

```python
def generate_transfer_flow(self, situation: str) -> str:
    relevant = self.search(situation, n_results=3)
    context = '\n\n'.join(relevant)
    prompt = f"""
    以下の参考情報をもとに、「{situation}」の転校手続きフローを作成してください。

    出力形式：
    ## 転校手続きフロー：{situation}

    ### ステップ一覧
    | ステップ | 内容 | 担当窓口 | 必要書類 | 期限・備考 |
    |---------|------|---------|---------|-----------|

    ### 補足事項
    - 注意点や特記事項を箇条書きで

    参考情報：{context}
    """
    return model.generate_content(prompt).text
```

**期待される出力例：**

```
## 転校手続きフロー：市内で引越しして小学校を転校する

### ステップ一覧

| ステップ | 内容 | 担当窓口 | 必要書類 | 期限・備考 |
|---------|------|---------|---------|-----------|
| 1 | 現在の学校に転校の旨を伝える | 在籍校（担任・教頭） | なし | できるだけ早めに |
| 2 | 転入届を提出 | 市民課（市役所1階） | 転出証明書（不要の場合も）・本人確認書類 | 引越し後14日以内 |
| 3 | 在学証明書・教科書給与証明書を受け取る | 在籍校 | なし | 転校前に必ず受け取る |
| 4 | 就学校指定変更申請を提出 | 学校教育課（市役所4階） | 在学証明書・教科書給与証明書 | 転入届後に手続き |
| 5 | 新しい学校で入学手続きをする | 新在籍校 | ステップ3の書類 | 学校の指示に従う |

### 補足事項
- 学期途中の転校も可能です。なるべく早めに学校教育課にご相談ください
- 教科書は新しい学校で無償で給与されます
- 給食費や学校諸費の精算が必要な場合があります（在籍校に確認）
- 就学援助を受けていた場合は新しい学校でも申請が必要です
```

---

### 機能2（発展）：入学準備チェックリスト生成

**実装のポイント：** 入学する学校種（小学校・中学校）と月（何月入学か）を受け取り、時系列で準備物をリスト化する。

```python
def generate_enrollment_checklist(self, school_type: str, entry_month: int = 4) -> str:
    """入学準備チェックリストを生成（オリジナル機能）"""
    relevant = self.search(f'{school_type} 入学 手続き', n_results=3)
    context = '\n\n'.join(relevant)
    month_before = entry_month - 1 if entry_month > 1 else 12

    prompt = f"""
    {school_type}に{entry_month}月から入学するお子さんの保護者向けに、
    入学準備チェックリストを時系列で作成してください。

    出力形式：
    ## {school_type}入学準備チェックリスト

    ### {month_before}月までに行うこと
    - [ ] 項目

    ### 入学式前日までに確認すること
    - [ ] 項目

    ### 入学後すぐに行うこと
    - [ ] 項目

    参考情報：{context}
    """
    return model.generate_content(prompt).text
```

**期待される出力例：**

```
## 小学校入学準備チェックリスト

### 3月までに行うこと
- [ ] 入学通知書を受け取る（10〜11月頃に市から郵送）
- [ ] 就学時健康診断に参加する（11月頃・指定された日時）
- [ ] 学校説明会に参加して準備物リストを受け取る（2〜3月）
- [ ] ランドセル・学用品を購入する（説明会後に具体的なリストが届く）
- [ ] 区域外就学を希望する場合は学校教育課へ申請する
- [ ] 就学援助制度が必要な場合は学校教育課または各学校へ相談する

### 入学式前日までに確認すること
- [ ] 入学通知書・健康診断の結果用紙を持参物に入れた
- [ ] 名前書きが必要な学用品はすべて記名した
- [ ] 通学路を子どもと一緒に歩いて安全を確認した
- [ ] 緊急連絡先・アレルギー情報等の書類を記入・準備した

### 入学後すぐに行うこと
- [ ] PTA・保護者会への入会手続き
- [ ] 給食費・学校諸費の引落口座登録
- [ ] 放課後の預かり（学童保育）が必要な場合は申請する
```

---

### Gradio UI 統合例（テーマD完成版）

```python
bot_d = EducationBot()

def chat_fn(message, history):
    return bot_d.answer(message)

def flow_fn(situation):
    if not situation.strip():
        return '状況を選択または入力してください'
    return bot_d.generate_transfer_flow(situation)

def checklist_fn(school_type, entry_month):
    return bot_d.generate_enrollment_checklist(school_type, int(entry_month))

with gr.Blocks(title='🏫 学校教育AIアシスタント', theme=gr.themes.Soft()) as demo:
    gr.Markdown('# 🏫 学校教育AIアシスタント')

    with gr.Tabs():
        with gr.Tab('💬 チャット相談'):
            gr.ChatInterface(fn=chat_fn, examples=[
                '子どもが小学校に入学するにはどうすればいいですか？',
                '市内で引越します。転校の手続きを教えてください',
                '就学援助制度について教えてください'
            ])

        with gr.Tab('🔄 転校フロー確認'):
            gr.Markdown('状況を選ぶと、転校手続きのステップを一覧表示します。')
            situation_in = gr.Dropdown(
                label='状況を選択',
                choices=[
                    '市内で引越しして小学校を転校する',
                    '他の市から転校してくる',
                    '他の市へ転校する',
                    '市内で引越しして中学校を転校する'
                ]
            )
            flow_out = gr.Markdown()
            gr.Button('フローを表示').click(flow_fn, inputs=situation_in, outputs=flow_out)

        with gr.Tab('📋 入学準備チェックリスト'):
            gr.Markdown('入学する学校と時期を選ぶと、準備チェックリストを生成します。')
            with gr.Row():
                school_in = gr.Dropdown(
                    label='学校の種類',
                    choices=['小学校', '中学校'],
                    value='小学校'
                )
                month_in = gr.Dropdown(
                    label='入学月',
                    choices=['4', '9'],
                    value='4'
                )
            checklist_out = gr.Markdown()
            gr.Button('チェックリストを生成').click(
                checklist_fn, inputs=[school_in, month_in], outputs=checklist_out
            )

demo.launch(share=True)
```

---

## 解答例の全体比較

| テーマ | 基本機能 | 発展機能 | GradioのUI工夫 |
|--------|---------|---------|--------------|
| A：住民窓口 | 必要書類チェックリスト生成 | 来庁タイミングアドバイス | タブ切替（チャット/チェックリスト/アドバイス） |
| B：土木・インフラ | 緊急度自動分類 + 報告票生成 | 報告番号の自動採番 | フォーム形式で入力 → 構造化報告票を出力 |
| C：福祉・子育て | 年齢別サービス一覧 | 申請期限リマインダー | スライダーで月齢入力 |
| D：教育 | 転校フロー生成 | 入学準備チェックリスト生成 | ドロップダウンで状況選択 |

### 共通する実装パターン

```
1. RAGで関連FAQを検索
   ↓
2. プロンプトで出力形式を厳密に指定（表・チェックリスト・JSON等）
   ↓
3. LLMの出力を受け取る
   ↓
4. 必要に応じて構造化（JSON解析・Markdown整形等）
   ↓
5. GradioのUIに渡す
```

> **設計のポイント：**  
> プロンプトで出力形式を具体的に指定するほど、LLMの出力がUIに組み込みやすくなります。  
> 「Markdown の表形式」「チェックボックス形式」「JSON形式のみ」など、用途に合わせて使い分けましょう。
