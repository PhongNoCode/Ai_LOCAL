import os
import pandas as pd
import requests
import ollama

# ==================== CẤU HÌNH HỆ THỐNG ====================
INPUT_FILE = 'input.csv'          # Tên file CSV đầu vào của bạn
MODEL_NAME = 'gemma:4b'           # Thay thế bằng tên chính xác của model Gemma bạn đã tải (ví dụ: gemma:2b, gemma:7b...)
EXTERNAL_AI_API_URL = 'https://api.external-ai.com/v1/check-url' # API của AI bên ngoài
EXTERNAL_AI_API_KEY = 'YOUR_API_KEY_HERE'                         # Key API của bạn
# ===========================================================

def load_data(file_path):
    """Đọc file CSV đầu vào"""
    if not os.path.exists(file_path):
        print(f"[!] Không tìm thấy file {file_path}. Vui lòng kiểm tra lại.")
        return None
    return pd.read_csv(file_path)

def send_to_local_ai(df, model_name):
    """Gửi toàn bộ dữ liệu vào Ollama Gemma để AI hiểu và phản hồi tổng quan"""
    print(f"\n--- 1. Đang gửi toàn bộ dữ liệu vào AI Local ({model_name}) ---")
    
    # Chuyển dataframe thành chuỗi văn bản để AI dễ đọc và phân tích
    data_str = df.to_string(index=False)
    
    prompt = f"""
Bạn là một chuyên gia phân tích an ninh mạng. Dưới đây là danh sách nhật ký truy cập hệ thống (bao gồm các cột username, time, risk, url):

{data_str}

Hãy đọc hiểu toàn bộ dữ liệu trên và đưa ra một nhận xét tổng quan ngắn gọn bằng tiếng Việt về tình hình dữ liệu này (ví dụ: tổng số lượt truy cập, những user đáng chú ý).
"""
    try:
        response = ollama.chat(model=model_name, messages=[
            {
                'role': 'user',
                'content': prompt,
            },
        ])
        print("\n[AI Local trả lời]:")
        print(response['message']['content'])
        print("-" * 50)
    except Exception as e:
        print(f"[Lỗi] Không thể kết nối hoặc gọi Ollama Local AI: {e}")
        print("Quá trình vẫn tiếp tục xử lý các bước sau...")

def call_external_ai_api(url):
    """
    Hàm gọi API đến AI bên ngoài để kiểm tra URL chưa có mã Risk=High.
    Hàm này đang để chế độ giả lập (mô phỏng), bạn hãy sửa lại theo đúng cấu trúc API thật của bạn.
    """
    headers = {
        "Authorization": f"Bearer {EXTERNAL_AI_API_KEY}",
        "Content-Type": "application/json"
    }
    payload = {"url": url}
    
    try:
        # BỎ COMMENT 3 DÒNG DƯỚI ĐÂY KHI BẠN RÁP API THẬT:
        # response = requests.post(EXTERNAL_AI_API_URL, json=payload, headers=headers)
        # result = response.json()
        # return "malicious" if result.get('is_dangerous') else "safe"
        
        # --- ĐOẠN MÔ PHỎNG (TEST) KHI CHƯA CÓ API THẬT ---
        import random
        # Giả lập ngẫu nhiên 70% an toàn (safe), 30% độc hại (malicious) để test code
        return "malicious" if random.random() < 0.3 else "safe"
        
    except Exception as e:
        print(f"    [Lỗi] Không thể gọi API AI ngoài cho URL {url}: {e}")
        return "error"

def save_to_user_file(row):
    """Lưu lại đúng 4 thông tin ban đầu vào file riêng của từng User"""
    username = str(row['username']).strip()
    user_file = f"user_{username}.csv"
    
    # Chuyển row hiện tại thành một DataFrame nhỏ
    df_row = pd.DataFrame([row])
    
    # Nếu file của user này đã có thì ghi tiếp vào (append), chưa có thì tạo mới kèm tiêu đề (header)
    if os.path.exists(user_file):
        df_row.to_csv(user_file, mode='a', header=False, index=False)
    else:
        df_row.to_csv(user_file, mode='w', header=True, index=False)

def save_to_list(url, list_type):
    """Ghi URL vào file blacklist.csv hoặc whitelist.csv mà không bị trùng lặp"""
    file_name = f"{list_type}.csv"
    df_url = pd.DataFrame([{"url": url}])
    
    if os.path.exists(file_name):
        # Đọc dữ liệu cũ lên để kiểm tra xem URL này đã nằm trong danh sách chưa
        existing_df = pd.read_csv(file_name)
        if url in existing_df['url'].values:
            return # Đã có rồi thì bỏ qua không ghi trùng
        df_url.to_csv(file_name, mode='a', header=False, index=False)
    else:
        df_url.to_csv(file_name, mode='w', header=True, index=False)

def main():
    # Bước 1: Đọc file dữ liệu đầu vào
    df = load_data(INPUT_FILE)
    if df is None:
        return
    
    # Chuẩn hóa tên cột để tránh lỗi viết hoa/viết thường
    df.columns = [col.strip().lower() for col in df.columns]
    required_columns = ['username', 'time', 'risk', 'url']
    if not all(col in df.columns for col in required_columns):
        print(f"[Lỗi] File CSV thiếu một trong các cột bắt buộc: {required_columns}")
        return

    # Bước 2: Đưa vào cho AI Local hiểu và đưa nhận xét
    send_to_local_ai(df, MODEL_NAME)
    
    print("\n--- 2. Bắt đầu quét phân loại chi tiết từng dòng dữ liệu ---")
    
    # Bước 3: Duyệt qua từng dòng để xử lý logic phân loại
    for index, row in df.iterrows():
        url = row['url']
        risk_value = str(row['risk']).strip().lower()
        
        # Hành động 1: Cứ có dòng đăng nhập nào là lưu ngay vào file riêng của User đó
        save_to_user_file(row)
        
        # Hành động 2: Kiểm tra cột Risk để phân phối URL
        if risk_value == 'high':
            print(f"[High Risk] URL: {url} -> Đã High sẵn, đưa thẳng vào Blacklist.")
            save_to_list(url, 'blacklist')
        else:
            print(f"[Check AI]  URL: {url} (Risk: {row['risk']}) -> Đang gọi AI bên ngoài...")
            ai_status = call_external_ai_api(url)
            
            if ai_status == "malicious":
                print(f"            => Kết quả: Nguy hiểm! Đưa vào Blacklist.")
                save_to_list(url, 'blacklist')
            elif ai_status == "safe":
                print(f"            => Kết quả: An toàn. Đưa vào Whitelist.")
                save_to_list(url, 'whitelist')
            else:
                print(f"            => Không thể xác định do lỗi API.")

    print("\n[HOÀN THÀNH] Tất cả các tệp tin CSV (user, blacklist, whitelist) đã được xuất thành công!")

if __name__ == "__main__":
    # ĐOẠN CODE NÀY ĐỂ TẠO FILE MẪU ĐỂ BẠN TEST THỬ (NẾU CHƯA CÓ FILE INPUT.CSV SẴN)
    if not os.path.exists(INPUT_FILE):
        print(f"Không tìm thấy file '{INPUT_FILE}', hệ thống tự động tạo 1 file mẫu để chạy thử...")
        sample_data = {
            'username': ['alex', 'bob', 'alex', 'charlie'],
            'time': ['2026-05-31 08:00', '2026-05-31 08:15', '2026-05-31 08:30', '2026-05-31 08:45'],
            'risk': ['high', 'low', 'medium', 'high'],
            'url': ['http://trang-web-doc-hai.com', 'https://google.com', 'http://link-nghi-van.net', 'http://virus-download.xyz']
        }
        pd.DataFrame(sample_data).to_csv(INPUT_FILE, index=False)
        
    main()
