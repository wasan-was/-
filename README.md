# -import streamlit as st
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import plotly.express as px
import os
import sys
from utils.scheduler import schedule_tasks
from utils.data_manager import load_tasks, save_tasks

# Page config
st.set_page_config(
    page_title="SmartScheduler | الجدولة الذكية",
    page_icon="📅",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Custom CSS
st.markdown("""
<style>
* {
  font-family: 'Cairo', sans-serif;
}
.main {
  background-color: #f4f4f9;
}
.stApp {
  direction: rtl;
}
.st-emotion-cache-16txtl3 h1, .st-emotion-cache-16txtl3 h2, .st-emotion-cache-16txtl3 h3 {
  text-align: right;
}
.main .block-container {
  padding-top: 2rem;
  padding-bottom: 2rem;
}
.st-bq {
  background-color: #e8f5e9;
  border-left: 5px solid #2e7d32;
}
</style>
""", unsafe_allow_html=True)

# Sidebar
with st.sidebar:
    st.title("SmartScheduler")
    st.subheader("جدولة المهام بذكاء")
    
    st.markdown("---")
    
    # User input for new task
    st.subheader("إضافة مهمة جديدة")
    task_name = st.text_input("اسم المهمة", key="task_name")
    task_description = st.text_area("وصف المهمة", key="task_description")
    task_duration = st.number_input("المدة المتوقعة (بالدقائق)", min_value=15, max_value=480, value=60, step=15, key="task_duration")
    task_priority = st.select_slider(
        "الأولوية",
        options=["منخفضة", "متوسطة", "عالية", "عاجلة"],
        value="متوسطة",
        key="task_priority"
    )
    
    # Due date
    today = datetime.now().date()
    task_due_date = st.date_input("تاريخ الاستحقاق", value=today + timedelta(days=1), key="task_due_date")
    
    if st.button("إضافة المهمة", key="add_task"):
        # Load existing tasks
        tasks = load_tasks()
        
        # Generate task ID
        task_id = len(tasks) + 1
        
        # Add new task
        new_task = {
            "id": task_id,
            "name": task_name,
            "description": task_description,
            "duration": task_duration,
            "priority": task_priority,
            "due_date": task_due_date.strftime("%Y-%m-%d"),
            "completed": False,
            "scheduled_time": None
        }
        
        tasks.append(new_task)
        save_tasks(tasks)
        st.success("تمت إضافة المهمة بنجاح!")
    
    st.markdown("---")
    
    # Navigation
    st.subheader("التنقل")
    page_options = {
        "الصفحة الرئيسية": "Home", 
        "جدول اليوم": "Today", 
        "إدارة المهام": "Tasks", 
        "الإحصائيات": "Stats"
    }
    selected_page = st.radio("اختر الصفحة", list(page_options.keys()), key="page_selector")
    current_page = page_options[selected_page]

# Main content area
st.title("نظام الجدولة الذكي")

if current_page == "Home":
    st.header("لوحة التحكم")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("المهام القادمة")
        tasks = load_tasks()
        upcoming_tasks = [task for task in tasks if not task["completed"] and task["due_date"] >= datetime.now().strftime("%Y-%m-%d")]
        
        if upcoming_tasks:
            for task in sorted(upcoming_tasks, key=lambda x: (x["due_date"], x["priority"])):
                priority_color = {
                    "منخفضة": "blue",
                    "متوسطة": "green",
                    "عالية": "orange",
                    "عاجلة": "red"
                }
                st.markdown(f"""
                <div style="padding: 10px; border-left: 5px solid {priority_color[task['priority']]}; margin-bottom: 10px; background-color: #f0f0f0;">
                    <h4 style="margin: 0;">{task['name']}</h4>
                    <p style="margin: 5px 0;">الأولوية: {task['priority']}</p>
                    <p style="margin: 5px 0;">موعد الاستحقاق: {task['due_date']}</p>
                </div>
                """, unsafe_allow_html=True)
        else:
            st.info("لا توجد مهام قادمة حالياً")
    
    with col2:
        st.subheader("إحصائيات سريعة")
        tasks = load_tasks()
        
        total_tasks = len(tasks)
        completed_tasks = len([task for task in tasks if task["completed"]])
        pending_tasks = total_tasks - completed_tasks
        
        st.metric("إجمالي المهام", total_tasks)
        st.metric("المهام المكتملة", completed_tasks)
        st.metric("المهام المعلقة", pending_tasks)
        
        if total_tasks > 0:
            completion_rate = (completed_tasks / total_tasks) * 100
            st.progress(completion_rate / 100)
            st.caption(f"معدل الإنجاز: {completion_rate:.1f}%")
    
    st.subheader("الجدولة الذكية")
    st.write("""
    يمكنك استخدام وظيفة الجدولة الذكية لتنظيم مهامك تلقائيًا بناءً على الأولوية والوقت المتاح.
    """)
    
    col1, col2 = st.columns(2)
    
    with col1:
        available_hours = st.slider("ساعات العمل المتاحة اليوم", min_value=1, max_value=12, value=8)
    
    with col2:
        if st.button("جدولة المهام", key="schedule_button"):
            tasks = load_tasks()
            scheduled_tasks = schedule_tasks(tasks, available_hours)
            save_tasks(scheduled_tasks)
            st.success("تمت جدولة المهام بنجاح!")

elif current_page == "Today":
    st.header("جدول اليوم")
    
    tasks = load_tasks()
    today_str = datetime.now().strftime("%Y-%m-%d")
    todays_tasks = [task for task in tasks if task["due_date"] == today_str or (task["scheduled_time"] and task["scheduled_time"].startswith(today_str))]
    
    if not todays_tasks:
        st.info("لا توجد مهام مجدولة لليوم")
    else:
        # Sort tasks by scheduled time if available
        todays_tasks.sort(key=lambda x: x.get("scheduled_time", ""))
        
        for i, task in enumerate(todays_tasks):
            with st.expander(f"{task['name']} - {task['priority']}", expanded=True):
                col1, col2 = st.columns([3, 1])
                
                with col1:
                    st.write(f"**الوصف:** {task['description']}")
                    st.write(f"**المدة المتوقعة:** {task['duration']} دقيقة")
                    if task.get("scheduled_time"):
                        st.write(f"**الوقت المجدول:** {task['scheduled_time'].split(' ')[1]}")
                
                with col2:
                    is_completed = st.checkbox("مكتملة", value=task["completed"], key=f"task_{task['id']}")
                    
                    if is_completed != task["completed"]:
                        task["completed"] = is_completed
                        save_tasks(tasks)
                        st.experimental_rerun()

elif current_page == "Tasks":
    st.header("إدارة المهام")
    
    tasks = load_tasks()
    
    # Task filtering
    st.subheader("تصفية المهام")
    col1, col2 = st.columns(2)
    
    with col1:
        filter_status = st.selectbox(
            "حالة المهمة",
            ["الكل", "مكتملة", "معلقة"],
            key="filter_status"
        )
    
    with col2:
        filter_priority = st.selectbox(
            "الأولوية",
            ["الكل", "منخفضة", "متوسطة", "عالية", "عاجلة"],
            key="filter_priority"
        )
    
    # Apply filters
    filtered_tasks = tasks
    if filter_status != "الكل":
        is_completed = filter_status == "مكتملة"
        filtered_tasks = [task for task in filtered_tasks if task["completed"] == is_completed]
    
    if filter_priority != "الكل":
        filtered_tasks = [task for task in filtered_tasks if task["priority"] == filter_priority]
    
    # Display tasks
    if not filtered_tasks:
        st.info("لا توجد مهام مطابقة للفلتر المحدد")
    else:
        for task in filtered_tasks:
            with st.expander(f"{task['name']} - {task['priority']}", expanded=False):
                col1, col2, col3 = st.columns([3, 1, 1])
                
                with col1:
                    st.write(f"**الوصف:** {task['description']}")
                    st.write(f"**موعد الاستحقاق:** {task['due_date']}")
                    st.write(f"**المدة المتوقعة:** {task['duration']} دقيقة")
                
                with col2:
                    is_completed = st.checkbox("مكتملة", value=task["completed"], key=f"manage_task_{task['id']}")
                    
                    if is_completed != task["completed"]:
                        task["completed"] = is_completed
                        save_tasks(tasks)
                        st.experimental_rerun()
                
                with col3:
                    if st.button("حذف", key=f"delete_task_{task['id']}"):
                        tasks.remove(task)
                        save_tasks(tasks)
                        st.success("تم حذف المهمة")
                        st.experimental_rerun()

elif current_page == "Stats":
    st.header("الإحصائيات والتحليلات")
    
    tasks = load_tasks()
    
    if not tasks:
        st.info("لا توجد بيانات كافية لعرض الإحصائيات")
    else:
        col1, col2 = st.columns(2)
        
        with col1:
            st.subheader("توزيع المهام حسب الأولوية")
            priority_counts = {}
            for task in tasks:
                priority = task["priority"]
                priority_counts[priority] = priority_counts.get(priority, 0) + 1
            
            fig = px.pie(
                values=list(priority_counts.values()),
                names=list(priority_counts.keys()),
                title="توزيع المهام حسب الأولوية",
                color_discrete_sequence=px.colors.sequential.Viridis
            )
            st.plotly_chart(fig, use_container_width=True)
        
        with col2:
            st.subheader("معدل إكمال المهام")
            completed = len([task for task in tasks if task["completed"]])
            pending = len(tasks) - completed
            
            fig = px.bar(
                x=["مكتملة", "معلقة"],
                y=[completed, pending],
                title="حالة المهام",
                color_discrete_sequence=["#2e7d32", "#c62828"]
            )
            st.plotly_chart(fig, use_container_width=True)
        
        # Task completion timeline
        st.subheader("الجدول الزمني للمهام")
        
        # Group tasks by due date
        due_dates = {}
        for task in tasks:
            due_date = task["due_date"]
            if due_date not in due_dates:
                due_dates[due_date] = {"total": 0, "completed": 0}
            
            due_dates[due_date]["total"] += 1
            if task["completed"]:
                due_dates[due_date]["completed"] += 1
        
        # Convert to DataFrame for plotting
        timeline_data = []
        for date, counts in due_dates.items():
            timeline_data.append({
                "التاريخ": date,
                "المهام": counts["total"],
                "المكتملة": counts["completed"],
                "المعلقة": counts["total"] - counts["completed"]
            })
        
        timeline_df = pd.DataFrame(timeline_data)
        timeline_df = timeline_df.sort_values("التاريخ")
        
        if not timeline_df.empty:
            fig = px.line(
                timeline_df,
                x="التاريخ",
                y=["المهام", "المكتملة", "المعلقة"],
                title="الجدول الزمني للمهام",
                markers=True
            )
            st.plotly_chart(fig, use_container_width=True)
        else:
            st.info("لا توجد بيانات كافية لعرض الجدول الزمني")

# This section allows running the app with app.run() style
if __name__ == "__main__":
    # Check if this file is being run directly
    if len(sys.argv) > 1 and sys.argv[0] == __file__:
        import streamlit.web.bootstrap as bootstrap
        from streamlit.web.server import Server
        
        class StreamlitApp:
            def run(self, host="0.0.0.0", port=3000):
                """Run the application on the specified host and port"""
                print(f"Starting SmartScheduler on http://{host}:{port}")
                # Set environment variables for Streamlit
                os.environ["STREAMLIT_SERVER_PORT"] = str(port)
                os.environ["STREAMLIT_SERVER_HEADLESS"] = "true"
                os.environ["STREAMLIT_SERVER_ADDRESS"] = host
                
                # Run the application
                bootstrap.run(__file__, "", [], {})
        
        # Create instance that can be imported and used with app.run()
        app = StreamlitApp()
        
        # Check if being called directly with arguments
        if len(sys.argv) > 1:
            try:
                # Parse command line arguments if provided
                if "--port" in sys.argv:
                    port_index = sys.argv.index("--port") + 1
                    if port_index < len(sys.argv):
                        port = int(sys.argv[port_index])
                    else:
                        port = 3000
                else:
                    port = 3000
                
                if "--host" in sys.argv:
                    host_index = sys.argv.index("--host") + 1
                    if host_index < len(sys.argv):
                        host = sys.argv[host_index]
                    else:
                        host = "0.0.0.0"
                else:
                    host = "0.0.0.0"
                
                # Start the app
                app.run(host=host, port=port)
            except Exception as e:
                print(f"Error starting application: {e}")
                # Fall back to normal Streamlit execution
                pass
                """
الجدولة الذكية للمهام بناءً على الأولوية والوقت المتاح
"""
from datetime import datetime, timedelta
import pandas as pd
import numpy as np
from typing import List, Dict, Any

# ترتيب قيم الأولوية من الأعلى إلى الأدنى
PRIORITY_RANKS = {
    "عاجلة": 4,
    "عالية": 3,
    "متوسطة": 2,
    "منخفضة": 1
}

def schedule_tasks(tasks: List[Dict[str, Any]], available_hours: int) -> List[Dict[str, Any]]:
    """
    جدولة المهام بناءً على الأولوية والوقت المتاح
    
    المعلمات:
        tasks (List[Dict]): قائمة بالمهام المراد جدولتها
        available_hours (int): عدد ساعات العمل المتاحة
    
    العائد:
        List[Dict]: قائمة المهام بعد تحديث أوقات الجدولة
    """
    # تحويل الساعات المتاحة إلى دقائق
    available_minutes = available_hours * 60
    
    # تصفية المهام غير المكتملة
    pending_tasks = [task for task in tasks if not task["completed"]]
    
    # ترتيب المهام حسب الأولوية وتاريخ الاستحقاق
    pending_tasks.sort(key=lambda x: (
        # ترتيب تنازلي حسب الأولوية
        -PRIORITY_RANKS[x["priority"]],
        # ترتيب تصاعدي حسب تاريخ الاستحقاق
        x["due_date"]
    ))
    
    # تحديد وقت البدء (8 صباحًا)
    start_time = datetime.now().replace(hour=8, minute=0, second=0, microsecond=0)
    current_time = start_time
    
    # جدولة المهام
    for task in pending_tasks:
        task_duration = task["duration"]  # المدة بالدقائق
        
        # تحقق مما إذا كان هناك وقت كافٍ متبقي
        if task_duration <= available_minutes:
            # تعيين وقت جدولة المهمة
            task["scheduled_time"] = current_time.strftime("%Y-%m-%d %H:%M")
            
            # تحديث الوقت الحالي والوقت المتاح
            current_time += timedelta(minutes=task_duration)
            available_minutes -= task_duration
        else:
            # لا يوجد وقت كاف للمهمة
            task["scheduled_time"] = None
    
    # إعادة المهام بعد الجدولة (تشمل المكتملة وغير المجدولة)
    return tasks

def get_optimal_schedule(tasks: List[Dict[str, Any]], work_hours: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    """
    إنشاء جدول أمثل استنادًا إلى المهام المتاحة وساعات العمل المحددة
    (وظيفة متقدمة للتوسع المستقبلي)
    
    المعلمات:
        tasks (List[Dict]): قائمة المهام
        work_hours (List[Dict]): ساعات العمل المحددة لكل يوم
        
    العائد:
        List[Dict]: جدول المهام المحسّن
    """
    # TODO: تنفيذ خوارزمية جدولة متقدمة باستخدام البرمجة الخطية أو خوارزميات التحسين
    
    # هذه نسخة مبسطة ستُستبدل بتنفيذ أكثر تقدمًا
    return schedule_tasks(tasks, 8)
    """
إدارة البيانات للتطبيق - تحميل وحفظ المهام
"""
import os
import json
from typing import List, Dict, Any
from datetime import datetime

# مسار ملف البيانات
DATA_DIR = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), "data")
TASKS_FILE = os.path.join(DATA_DIR, "tasks.json")

def load_tasks() -> List[Dict[str, Any]]:
    """
    تحميل المهام من ملف JSON
    
    العائد:
        List[Dict]: قائمة المهام
    """
    # التأكد من وجود مجلد البيانات
    if not os.path.exists(DATA_DIR):
        os.makedirs(DATA_DIR)
    
    # التحقق من وجود ملف المهام
    if not os.path.exists(TASKS_FILE):
        return []
    
    try:
        with open(TASKS_FILE, 'r', encoding='utf-8') as file:
            tasks = json.load(file)
            return tasks
    except (json.JSONDecodeError, FileNotFoundError):
        # في حالة تلف الملف أو عدم وجوده، إعادة قائمة فارغة
        return []

def save_tasks(tasks: List[Dict[str, Any]]) -> bool:
    """
    حفظ المهام إلى ملف JSON
    
    المعلمات:
        tasks (List[Dict]): قائمة المهام المراد حفظها
        
    العائد:
        bool: نجاح أو فشل عملية الحفظ
    """
    # التأكد من وجود مجلد البيانات
    if not os.path.exists(DATA_DIR):
        os.makedirs(DATA_DIR)
    
    try:
        with open(TASKS_FILE, 'w', encoding='utf-8') as file:
            json.dump(tasks, file, ensure_ascii=False, indent=2)
        return True
    except Exception as e:
        print(f"خطأ في حفظ المهام: {str(e)}")
        return False

def export_tasks_to_csv(filename: str = None) -> str:
    """
    تصدير المهام إلى ملف CSV
    
    المعلمات:
        filename (str, optional): اسم الملف الذي سيتم التصدير إليه
        
    العائد:
        str: مسار الملف المصدر
    """
    import pandas as pd
    
    if filename is None:
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"tasks_export_{timestamp}.csv"
    
    export_path = os.path.join(DATA_DIR, filename)
    
    tasks = load_tasks()
    if not tasks:
        return ""
    
    # تحويل البيانات إلى DataFrame
    df = pd.DataFrame(tasks)
    
    # تصدير البيانات
    df.to_csv(export_path, index=False, encoding='utf-8-sig')
    
    return export_path

def import_tasks_from_csv(file_path: str) -> bool:
    """
    استيراد المهام من ملف CSV
    
    المعلمات:
        file_path (str): مسار ملف CSV
        
    العائد:
        bool: نجاح أو فشل الاستيراد
    """
    import pandas as pd
    
    try:
        # قراءة البيانات من الملف
        df = pd.read_csv(file_path)
        
        # تحويل DataFrame إلى قائمة من القواميس
        tasks = df.to_dict('records')
        
        # حفظ المهام المستوردة
        save_tasks(tasks)
        
        return True
    except Exception as e:
        print(f"خطأ في استيراد المهام: {str(e)}")
        return False
        import streamlit as st
import os
import pandas as pd
from datetime import datetime
from utils.data_manager import load_tasks, save_tasks, export_tasks_to_csv, import_tasks_from_csv

# Page config
st.set_page_config(
    page_title="الإعدادات | SmartScheduler",
    page_icon="⚙️",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Custom CSS
st.markdown("""
<style>
* {
  font-family: 'Cairo', sans-serif;
}
.main {
  background-color: #f4f4f9;
}
.stApp {
  direction: rtl;
}
.st-emotion-cache-16txtl3 h1, .st-emotion-cache-16txtl3 h2, .st-emotion-cache-16txtl3 h3 {
  text-align: right;
}
.main .block-container {
  padding-top: 2rem;
  padding-bottom: 2rem;
}
</style>
""", unsafe_allow_html=True)

# Main content
st.title("⚙️ إعدادات التطبيق")

st.write("""
يمكنك من خلال هذه الصفحة ضبط إعدادات التطبيق وإدارة البيانات.
""")

# Tabs for different settings categories
tab1, tab2, tab3 = st.tabs(["تصدير واستيراد البيانات", "إعدادات العرض", "معلومات التطبيق"])

with tab1:
    st.header("تصدير واستيراد البيانات")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("تصدير البيانات")
        st.write("يمكنك تصدير جميع المهام الخاصة بك إلى ملف CSV للنسخ الاحتياطي أو التحليل.")
        
        export_format = st.radio("اختر تنسيق التصدير:", ["CSV", "JSON"], horizontal=True)
        
        if st.button("تصدير البيانات", key="export_button"):
            tasks = load_tasks()
            
            if not tasks:
                st.warning("لا توجد بيانات للتصدير!")
            else:
                if export_format == "CSV":
                    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
                    export_filename = f"smart_scheduler_export_{timestamp}.csv"
                    export_path = export_tasks_to_csv(export_filename)
                    
                    if export_path:
                        # Read file content for download
                        with open(export_path, "r", encoding="utf-8-sig") as file:
                            file_content = file.read()
                        
                        st.download_button(
                            label="تنزيل ملف التصدير",
                            data=file_content,
                            file_name=export_filename,
                            mime="text/csv"
                        )
                else:  # JSON
                    import json
                    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
                    export_filename = f"smart_scheduler_export_{timestamp}.json"
                    
                    json_str = json.dumps(tasks, ensure_ascii=False, indent=2)
                    
                    st.download_button(
                        label="تنزيل ملف التصدير",
                        data=json_str,
                        file_name=export_filename,
                        mime="application/json"
                    )
    
    with col2:
        st.subheader("استيراد البيانات")
        st.write("يمكنك استيراد المهام من ملف CSV أو JSON.")
        
        uploaded_file = st.file_uploader("اختر ملف المهام للاستيراد", type=["csv", "json"])
        
        if uploaded_file is not None:
            try:
                file_extension = uploaded_file.name.split(".")[-1].lower()
                
                if file_extension == "csv":
                    # Process CSV upload
                    df = pd.read_csv(uploaded_file)
                    tasks = df.to_dict('records')
                    save_tasks(tasks)
                    st.success(f"تم استيراد {len(tasks)} مهمة بنجاح!")
                
                elif file_extension == "json":
                    # Process JSON upload
                    import json
                    tasks = json.loads(uploaded_file.getvalue().decode("utf-8"))
                    save_tasks(tasks)
                    st.success(f"تم استيراد {len(tasks)} مهمة بنجاح!")
                
            except Exception as e:
                st.error(f"حدث خطأ أثناء استيراد الملف: {str(e)}")
    
    st.markdown("---")
    
    st.subheader("إدارة البيانات")
    
    danger_zone = st.expander("⚠️ منطقة الخطر", expanded=False)
    
    with danger_zone:
        st.warning("العمليات التالية لا يمكن التراجع عنها. يرجى الحذر!")
        
        if st.button("حذف جميع المهام", key="delete_all_tasks"):
            confirm = st.text_input("اكتب 'تأكيد' للمتابعة:")
            
            if confirm.strip() == "تأكيد":
                save_tasks([])
                st.success("تم حذف جميع المهام بنجاح!")
                st.experimental_rerun()

with tab2:
    st.header("إعدادات العرض")
    
    st.subheader("المظهر العام")
    
    theme_color = st.color_picker(
        "اللون الرئيسي للتطبيق:",
        "#2e7d32",
        key="theme_color"
    )
    
    # Save theme preferences to session state
    if "theme_settings" not in st.session_state:
        st.session_state.theme_settings = {
            "primary_color": theme_color
        }
    else:
        st.session_state.theme_settings["primary_color"] = theme_color
    
    st.write("تم تطبيق اللون:", theme_color)
    
    # Apply custom theme to the current page
    st.markdown(f"""
    <style>
        /* تغيير لون الأزرار */
        .stButton > button {{
            background-color: {theme_color};
            color: white;
        }}
        
        /* تغيير لون العناوين */
        h1, h2, h3 {{
            color: {theme_color};
        }}
    </style>
    """, unsafe_allow_html=True)
    
    st.subheader("خيارات التقويم")
    
    calendar_start_day = st.selectbox(
        "يبدأ الأسبوع من يوم:",
        ["السبت", "الأحد", "الاثنين"],
        index=0,
        key="calendar_start_day"
    )
    
    display_completed = st.checkbox(
        "عرض المهام المكتملة في التقويم",
        value=True,
        key="display_completed"
    )
    
    # Save calendar preferences to session state
    if "calendar_settings" not in st.session_state:
        st.session_state.calendar_settings = {
            "start_day": calendar_start_day,
            "show_completed": display_completed
        }
    else:
        st.session_state.calendar_settings["start_day"] = calendar_start_day
        st.session_state.calendar_settings["show_completed"] = display_completed

with tab3:
    st.header("معلومات التطبيق")
    
    st.markdown("""
    ## SmartScheduler | الجدولة الذكية
    
    **الإصدار:** 1.0.0
    
    **وصف التطبيق:**
    تطبيق ذكاء ا
