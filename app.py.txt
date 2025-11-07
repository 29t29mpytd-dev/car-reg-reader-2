
# --------------------------------------------------------
# Car Registration Extractor Web App (Streamlit + EasyOCR)
# --------------------------------------------------------

import streamlit as st
import easyocr
import pandas as pd
import tempfile
import os

st.set_page_config(page_title="Car Registration Reader", page_icon="🚗", layout="wide")
st.title("🚗 Car Registration Reader")
st.write("Upload one or more car photos — this app will detect and extract registration numbers automatically.")

# Create OCR reader once
@st.cache_resource
def load_reader():
    return easyocr.Reader(['en'])

reader = load_reader()

uploaded_files = st.file_uploader("Upload car images", type=["jpg", "jpeg", "png"], accept_multiple_files=True)

if uploaded_files:
    st.info(f"Processing {len(uploaded_files)} image(s)... please wait.")
    results = []

    progress = st.progress(0)
    total = len(uploaded_files)

    for i, file in enumerate(uploaded_files):
        # Save file temporarily
        temp = tempfile.NamedTemporaryFile(delete=False)
        temp.write(file.read())
        temp_path = temp.name

        detections = reader.readtext(temp_path)
        reg = None

        for _, text, conf in detections:
            text_clean = text.replace(" ", "").upper()
            if len(text_clean) >= 5 and any(c.isdigit() for c in text_clean):
                reg = text_clean
                break

        results.append({"filename": file.name, "registration": reg or "Not detected"})

        progress.progress((i + 1) / total)
        temp.close()
        os.remove(temp_path)

    st.success("✅ Done!")

    df = pd.DataFrame(results)
    st.subheader("Detected Registrations")
    st.dataframe(df, use_container_width=True)

    # Download button
    csv = df.to_csv(index=False).encode('utf-8')
    st.download_button(
        label="📥 Download CSV",
        data=csv,
        file_name="car_registrations.csv",
        mime="text/csv",
    )
