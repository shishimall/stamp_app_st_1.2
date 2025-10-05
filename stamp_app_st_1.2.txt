# stamp_app_st

# stamp_app_st_1.2.py
# ver.1.2：日本語フォント同梱／ローカル＆Streamlit Cloud両対応版

import streamlit as st
from PIL import Image, ImageDraw, ImageFont
from datetime import datetime
import io, re, os

# ------------------------------------------------------------
# 共通関数：Pillow10対応 テキストサイズ取得
# ------------------------------------------------------------
def text_size(draw, text, font):
    bbox = draw.textbbox((0, 0), text, font=font)
    return bbox[2] - bbox[0], bbox[3] - bbox[1]


# ------------------------------------------------------------
# 日付入力補助関数
# ------------------------------------------------------------
def normalize_date_input(input_str):
    """ユーザー入力を yy.mm.dd 形式に正規化する"""
    if not input_str or input_str.strip() == "":
        return datetime.now().strftime("%y.%m.%d")

    s = re.sub(r"[^\d]", "", input_str)
    if len(s) == 6:  # yyMMdd
        return f"{s[0:2]}.{s[2:4]}.{s[4:6]}"
    elif len(s) == 8:  # yyyyMMdd
        return f"{s[2:4]}.{s[4:6]}.{s[6:8]}"
    else:
        return datetime.now().strftime("%y.%m.%d")


# ------------------------------------------------------------
# フォントパス自動判定（Windows / Linux / 同梱フォント対応）
# ------------------------------------------------------------
def get_font_path(font_family="msgothic.ttc"):
    """環境に応じて最適なフォントパスを返す（同梱フォント優先）"""
    # 優先：同梱日本語フォント
    local_font = os.path.join(os.path.dirname(__file__), "fonts", "NotoSansJP-Regular.ttf")
    if os.path.exists(local_font):
        return local_font

    # Windows / Linux 共通候補
    candidates = [
        f"C:/Windows/Fonts/{font_family}",
        "C:/Windows/Fonts/meiryo.ttc",
        "C:/Windows/Fonts/msmincho.ttc",
        "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
        "/usr/share/fonts/truetype/liberation/LiberationSans-Regular.ttf",
        "/usr/share/fonts/truetype/freefont/FreeSans.ttf",
    ]
    for path in candidates:
        if os.path.exists(path):
            return path

    raise FileNotFoundError("利用可能なフォントが見つかりません。")


# ------------------------------------------------------------
# 印影生成関数
# ------------------------------------------------------------
def create_stamp(name_top="宮", name_bottom="原",
                 include_date=True, custom_date=None,
                 font_family="msgothic.ttc",
                 font_size=80, top_offset=-150, bottom_offset=80,
                 scale_factor=1.0):
    """印影生成メイン関数"""
    size = 400
    img = Image.new("RGBA", (size, size), (255, 255, 255, 0))
    draw = ImageDraw.Draw(img)
    center = size / 2
    red = (220, 0, 0, 255)

    # 外枠（円）
    draw.ellipse((20, 20, size - 20, size - 20), outline=red, width=10)

    # ✅ フォントパスを自動判定（Cloudでも日本語OK）
    font_path = get_font_path(font_family)
    font = ImageFont.truetype(font_path, int(font_size * scale_factor))
    font_small = ImageFont.truetype(font_path, int(font_size * 0.6))

    # 上段（名字上）
    w1, h1 = text_size(draw, name_top, font)
    draw.text((center - w1 / 2, center + top_offset), name_top, fill=red, font=font)

    # 日付印モードのみ中央の日付と横線を描画
    if include_date:
        date_str = normalize_date_input(custom_date)
        w2, h2 = text_size(draw, date_str, font_small)
        draw.text((center - w2 / 2, center - h2 / 2 + 5), date_str, fill=red, font=font_small)

        margin = 35
        draw.line((margin, center - 45, size - margin, center - 45), fill=red, width=3)
        draw.line((margin, center + 60, size - margin, center + 60), fill=red, width=3)

    # 下段
    w3, h3 = text_size(draw, name_bottom, font)
    draw.text((center - w3 / 2, center + bottom_offset), name_bottom, fill=red, font=font)

    return img


# ------------------------------------------------------------
# Streamlit アプリ本体
# ------------------------------------------------------------
st.set_page_config(page_title="印影ジェネレーター", layout="wide")

st.title("🟥 電子日付印／シャチハタ印ジェネレーター（ver.1.2 日本語フォント対応）")

col_preview, col_controls = st.columns([2, 1])

with col_controls:
    st.subheader("🛠 調整パネル")

    # 名前入力
    name_top = st.text_input("上段（例：宮）", value="宮")
    name_bottom = st.text_input("下段（例：原／印）", value="原")

    # 印影モード選択
    mode = st.radio(
        "印影タイプ",
        ["日付印タイプ（2本線あり）", "シャチハタ風（中央寄せ・日付なし）"],
        index=0
    )

    # フォント選択（ローカルでは指定、Cloudでは自動代替）
    font_family = st.selectbox(
        "フォントを選択（CloudではNotoSansJP-Regularを自動使用）",
        ["msgothic.ttc", "meiryo.ttc", "msmincho.ttc"]
    )

    # ✅ モード別自動補正値
    if mode == "シャチハタ風（中央寄せ・日付なし）":
        include_date = False
        base_size_default = 120
        top_offset_default = -140
        bottom_offset_default = 15
        custom_date = None
    else:
        include_date = True
        base_size_default = 80
        top_offset_default = -150
        bottom_offset_default = 80
        custom_date = st.text_input(
            "日付を指定（例：25.10.05, 2025-10-05, 空欄で今日）",
            value=datetime.now().strftime("%y.%m.%d")
        )

    st.markdown("### 文字サイズ・位置調整")
    base_size = st.slider("基本フォントサイズ", 40, 140, base_size_default)
    scale_factor = st.slider("拡大倍率（全体）", 0.8, 1.5, 1.0, 0.05)

    st.markdown("#### 上下の位置オフセット（中心基準）")
    top_offset = st.slider("上段の上下位置", -200, 0, top_offset_default, 5)
    bottom_offset = st.slider("下段の上下位置", 0, 200, bottom_offset_default, 5)

    st.markdown("---")
    st.caption("※シャチハタ風は漢字2文字に最適化された配置値（120/-140/+15）を自動設定します。")


# 🔸 即時プレビュー生成（ボタン不要）
img = create_stamp(
    name_top, name_bottom,
    include_date,
    custom_date,
    font_family,
    base_size,
    top_offset,
    bottom_offset,
    scale_factor
)

with col_preview:
    st.image(img, caption="印影プレビュー（背景透過PNG・日本語フォント対応）", use_column_width=False)

    buf = io.BytesIO()
    img.save(buf, format="PNG")

    st.download_button(
        "📥 PNGを保存",
        data=buf.getvalue(),
        file_name=f"{name_top}{name_bottom}_stamp.png",
        mime="image/png"
    )
