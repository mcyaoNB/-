# -
找家里东西专用
import os
import openai
import pandas as pd
from datetime import datetime

class LocationDatabase:
    def __init__(self):
        self.locations = {}  # 内存中的位置字典
        self.load_data()     # 初始化时加载数据
        
    def load_data(self, filepath='locations.txt'):
        """从文件加载数据，支持txt和excel"""
        if filepath.endswith('.txt'):
            if os.path.exists(filepath):
                with open(filepath, 'r') as f:
                    for line in f:
                        item, location = line.strip().split(',')
                        self.locations[item] = location
                        
        elif filepath.endswith(('.xlsx', '.xls')):
            df = pd.read_excel(filepath)
            self.locations = df.set_index('Item')['Location'].to_dict()

    def save_data(self, filepath='locations.txt'):
        """保存数据到文件"""
        if filepath.endswith('.txt'):
            with open(filepath, 'w') as f:
                for item, loc in self.locations.items():
                    f.write(f"{item},{loc}\n")
                    
        elif filepath.endswith(('.xlsx', '.xls')):
            df = pd.DataFrame(list(self.locations.items()), columns=['Item', 'Location'])
            df.to_excel(filepath, index=False)

    def add_location(self, item, location):
        """添加/更新物品位置"""
        self.locations[item] = location
        self.save_data()  # 每次修改自动保存
        
    def get_location(self, item):
        """查询物品位置"""
        return self.locations.get(item, "未找到该物品")

class AIAssistant:
    def __init__(self, db):
        self.db = db
        openai.api_key = 'YOUR_API_KEY'  # 替换为你的OpenAI API密钥
        
    def parse_command(self, text):
        """使用OpenAI解析用户指令"""
        prompt = f"""
        请解析以下家居物品位置指令，返回JSON格式：
        {{
            "action": "query|update",
            "item": "物品名称",
            "location": "位置信息（如果是更新操作）"
        }}
        
        用户输入：{text}
        """
        
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": prompt}]
        )
        
        try:
            return eval(response.choices[0].message.content)
        except:
            return {"action": "error"}

    def process_command(self, text):
        """处理用户指令"""
        result = self.parse_command(text)
        
        if result['action'] == 'query':
            location = self.db.get_location(result['item'])
            return f"{result['item']}的位置是：{location}"
            
        elif result['action'] == 'update':
            self.db.add_location(result['item'], result['location'])
            return f"已更新{result['item']}的位置到：{result['location']}"
            
        return "未能理解您的指令，请重新表述"

# 使用示例
if __name__ == "__main__":
    db = LocationDatabase()
    assistant = AIAssistant(db)
    
    while True:
        user_input = input("\n请输入指令（输入exit退出）: ")
        if user_input.lower() == 'exit':
            break
            
        response = assistant.process_command(user_input)
        print("AI助手:", response)
        
    # 退出时保存数据到excel备份
    db.save_data(f"location_backup_{datetime.now().strftime('%Y%m%d')}.xlsx")
