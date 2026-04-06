# -[app.py](https://github.com/user-attachments/files/26505795/app.py)
import streamlit as st
import pandas as pd
import plotly.express as px
from io import BytesIO

st.set_page_config(
    page_title="청주고등학교 출결 분석",
    page_icon="🎓",
    layout="wide"
)

# 🎨 스타일 + 헤더 (🔥 반드시 markdown + unsafe_allow_html=True)
st.markdown("""
<style>
.block-container {
    padding-top: 2rem;
    padding-left: 3rem;
    padding-right: 3rem;
}

/* 헤더 */
.header {
    display: flex;
    align-items: center;
    gap: 14px;
    margin-bottom: 10px;
}

.logo {
    width: 48px;
    height: 48px;
    border-radius: 50%;
    background: #2563EB;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
    font-weight: 700;
    font-size: 20px;
}

.title {
    font-size: 20px;
    font-weight: 700;
}

.subtitle {
    font-size: 14px;
    color: #6b7280;
}

.divider {
    margin-top: 10px;
    margin-bottom: 25px;
    border-top: 1px solid #e5e7eb;
}

/* 카드 */
.card {
    padding: 24px;
    border-radius: 18px;
    background: linear-gradient(135deg, #ffffff, #eef2ff);
    box-shadow: 0 6px 18px rgba(0,0,0,0.08);
    text-align: center;
}

.big {
    font-size: 28px;
    font-weight: 700;
}

.label {
    font-size: 13px;
    color: #6b7280;
}

.dot {
    display: inline-block;
    width: 10px;
    height: 10px;
    border-radius: 50%;
    margin-right: 6px;
}

.blue { background-color: #2563EB; }
.green { background-color: #059669; }
.red { background-color: #DC2626; }
.orange { background-color: #F59E0B; }
</style>

<div class="header">
    <div class="logo">3</div>
    <div>
        <div class="title">청주고등학교 3학년</div>
        <div class="subtitle">자율학습 출결 분석 시스템</div>
    </div>
</div>

<div class="divider"></div>
""", unsafe_allow_html=True)

# 📂 사이드바
st.sidebar.markdown("### 🎓 청주고 3학년")
st.sidebar.markdown("출결 관리 시스템")
st.sidebar.markdown("---")

files1 = st.sidebar.file_uploader("출결통계 (여러 개)", type=["xlsx"], accept_multiple_files=True)
files2 = st.sidebar.file_uploader("출결기록 (여러 개)", type=["xlsx"], accept_multiple_files=True)

if files1 and files2:

    # 📊 데이터 병합
    df1 = pd.concat([pd.read_excel(f) for f in files1], ignore_index=True)
    df2 = pd.concat([pd.read_excel(f) for f in files2], ignore_index=True)

    # 🧠 전처리
    df1["학번"] = df1["학번"].astype(str).str.zfill(5)
    df2["학번"] = df2["학번"].astype(str).str.zfill(5)

    df1["반번호"] = df1["학번"].str[1:3].astype(int)
    df2["반번호"] = df2["학번"].str[1:3].astype(int)

    df1 = df1[df1["반번호"] != 0]
    df2 = df2[df2["반번호"] != 0]

    df1["학급"] = df1["반번호"].astype(str) + "반"
    df2["학급"] = df2["반번호"].astype(str) + "반"

    df1["출석률"] = df1["출석률"].astype(str).str.replace("%", "").astype(float)

    # 결석 횟수
    absence_count = df2.groupby("이름").size().reset_index(name="결석횟수")
    df1 = df1.merge(absence_count, on="이름", how="left")
    df1["결석횟수"] = df1["결석횟수"].fillna(0)

    # 🔍 필터
    st.sidebar.header("🔍 필터")

    class_list = sorted(df1["반번호"].unique())
    class_labels = [f"{i}반" for i in class_list]

    selected_class = st.sidebar.selectbox("학급", ["전체"] + class_labels)
    search_name = st.sidebar.text_input("이름 검색")

    filtered_df1 = df1.copy()
    filtered_df2 = df2.copy()

    if selected_class != "전체":
        num = int(selected_class.replace("반", ""))
        filtered_df1 = filtered_df1[filtered_df1["반번호"] == num]
        filtered_df2 = filtered_df2[filtered_df2["반번호"] == num]

    if search_name:
        filtered_df1 = filtered_df1[filtered_df1["이름"].str.contains(search_name)]
        filtered_df2 = filtered_df2[filtered_df2["이름"].str.contains(search_name)]

    # 📊 KPI
    col1, col2, col3, col4 = st.columns(4)

    avg = round(filtered_df1["출석률"].mean(), 1) if len(filtered_df1) else 0
    total_abs = int(filtered_df1["결석횟수"].sum())
    danger = len(filtered_df1[filtered_df1["출석률"] <= 70])

    col1.markdown(f"<div class='card'><div class='label'><span class='dot blue'></span>학생 수</div><div class='big'>{len(filtered_df1)}</div></div>", unsafe_allow_html=True)
    col2.markdown(f"<div class='card'><div class='label'><span class='dot green'></span>평균 출석률</div><div class='big'>{avg}%</div></div>", unsafe_allow_html=True)
    col3.markdown(f"<div class='card'><div class='label'><span class='dot red'></span>총 결석</div><div class='big'>{total_abs}</div></div>", unsafe_allow_html=True)
    col4.markdown(f"<div class='card'><div class='label'><span class='dot orange'></span>위험 학생</div><div class='big'>{danger}</div></div>", unsafe_allow_html=True)

    st.markdown("---")

    # 📥 다운로드
    def to_excel(df):
        output = BytesIO()
        with pd.ExcelWriter(output, engine='openpyxl') as writer:
            df.to_excel(writer, index=False)
        return output.getvalue()

    st.download_button("📥 결과 다운로드", to_excel(filtered_df1), "출결분석.xlsx", use_container_width=True)

    # 📊 그래프
    st.subheader("📊 반별 평균 출석률")

    class_avg = df1.groupby("반번호")["출석률"].mean().reset_index().sort_values("반번호")
    class_avg["학급"] = class_avg["반번호"].astype(str) + "반"

    class_avg["상태"] = class_avg["출석률"].apply(
        lambda x: "위험" if x <= 70 else "보통" if x <= 85 else "양호"
    )

    fig = px.bar(
        class_avg,
        x="학급",
        y="출석률",
        color="상태",
        category_orders={"학급": class_avg["학급"].tolist()},
        color_discrete_map={
            "위험": "#DC2626",
            "보통": "#F59E0B",
            "양호": "#059669"
        }
    )

    st.plotly_chart(fig, use_container_width=True)

    # 🚨 위험 학생
    st.subheader("🚨 출석률 70% 이하")
    st.dataframe(filtered_df1[filtered_df1["출석률"] <= 70])

    # 👤 상세
    st.subheader("📋 학생 상세")

    for _, row in filtered_df1.sort_values("출석률").iterrows():

        label = f"{row['이름']} ({row['출석률']}%)"
        if row["출석률"] <= 70:
            label = f"🔴 {label}"
        elif row["출석률"] <= 85:
            label = f"🟠 {label}"

        with st.expander(label):
            st.write(f"학번: {row['학번']}")
            st.write(f"학급: {row['학급']}")
            st.write(f"결석 횟수: {int(row['결석횟수'])}")

            data = filtered_df2[filtered_df2["이름"] == row["이름"]]

            if len(data):
                st.dataframe(data[["날짜", "교시"]])
            else:
                st.success("결석 없음 🎉")

else:
    st.info("👈 왼쪽에서 파일 업로드")

st.caption("청주고등학교 3학년 출결 분석 시스템 vFinal 🚀")
