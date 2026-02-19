import streamlit as st
import pandas as pd
import requests
import io

# Page Configuration
st.set_page_config(page_title="SKU URL Verifier", layout="wide")

def check_sku_in_url(sku, url):
    """
    URL ke content me SKU ko verify karne ke liye function.
    """
    try:
        # Request headers to mimic a browser
        headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
        }
        response = requests.get(url, headers=headers, timeout=10)
        
        if response.status_code == 200:
            # Check if SKU exists in the page text
            if str(sku).lower() in response.text.lower():
                return "Correct"
            else:
                return "Incorrect"
        else:
            return f"Error: Status {response.status_code}"
    except Exception as e:
        return f"Error: {str(e)}"

def main():
    st.title("📦 SKU Verification Tool")
    st.write("Apni CSV file upload karein jisme 'sku' aur 'url' columns hon.")

    # File uploader
    uploaded_file = st.file_uploader("CSV ya Excel file select karein", type=["csv", "xlsx"])

    if uploaded_file is not None:
        try:
            # File reading based on extension
            if uploaded_file.name.endswith('.csv'):
                df = pd.read_csv(uploaded_file)
            else:
                df = pd.read_excel(uploaded_file)

            # Column validation
            required_cols = ['sku', 'url']
            if not all(col in df.columns.map(str.lower) for col in required_cols):
                st.error("File me 'sku' aur 'url' naam ke columns hona zaroori hai!")
                return

            # Normalize column names to lowercase for easy access
            df.columns = [c.lower() for c in df.columns]

            st.write("Data Preview:")
            st.dataframe(df.head())

            if st.button("Verification Start Karein"):
                results = []
                progress_bar = st.progress(0)
                status_text = st.empty()

                for index, row in df.iterrows():
                    sku = row['sku']
                    url = row['url']
                    
                    status_text.text(f"Checking SKU: {sku} at URL...")
                    status = check_sku_in_url(sku, url)
                    results.append(status)
                    
                    # Update progress
                    progress_bar.progress((index + 1) / len(df))

                df['verification_result'] = results
                status_text.success("Verification Complete!")

                # Function to style the dataframe for display
                def color_result(val):
                    color = 'green' if val == 'Correct' else 'red'
                    return f'color: {color}; font-weight: bold'

                st.write("Results:")
                st.dataframe(df.style.applymap(color_result, subset=['verification_result']))

                # Download processed file
                output = io.BytesIO()
                with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
                    df.to_excel(writer, index=False, sheet_name='Verification_Results')
                
                processed_data = output.getvalue()
                
                st.download_button(
                    label="Verified File Download Karein",
                    data=processed_data,
                    file_name="verified_sku_data.xlsx",
                    mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
                )

        except Exception as e:
            st.error(f"File processing me error: {e}")

if __name__ == "__main__":
    main()
