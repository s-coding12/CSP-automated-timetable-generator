import pandas as pd
import numpy as np
import streamlit as st
import time
from datetime import datetime

st.set_page_config(page_title="Timetable System", page_icon="📚", layout="wide")

st.markdown("""
<style>
    .main {
        background: linear-gradient(45deg, #FF6B6B, #FF8E53, #FFD166, #06D6A0, #118AB2, #073B4C, #7209B7, #F72585, #3A0CA3);
        background-size: 400% 400%;
        animation: gradient 15s ease infinite;
        color: #000000;
        font-weight: bold;
    }
    @keyframes gradient {
        0% { background-position: 0% 50% }
        50% { background-position: 100% 50% }
        100% { background-position: 0% 50% }
    }
    .stApp { background: transparent; }
    .css-1d391kg, .css-1lcbmhc, .css-1outwn7 {
        background-color: rgba(255, 255, 255, 0.95);
        border-radius: 15px;
        padding: 20px;
        margin: 10px 0;
        border: 3px solid #000000;
    }
    .stButton>button {
        background: linear-gradient(45deg, #FF6B6B, #4ECDC4);
        color: black;
        font-weight: bold;
        border: 2px solid #000000;
        border-radius: 10px;
        padding: 10px 20px;
        font-size: 16px;
    }
    .stButton>button:hover {
        background: linear-gradient(45deg, #FF8E53, #118AB2);
        transform: scale(1.05);
    }
    h1, h2, h3, h4, h5, h6 { color: #000000; font-weight: bold; }
    .stDataFrame { background-color: rgba(255, 255, 255, 0.98); border-radius: 10px; border: 2px solid #000000; }
    .stExpander { background-color: rgba(255, 255, 255, 0.95); border: 2px solid #000000; border-radius: 10px; margin: 10px 0; }
</style>
""", unsafe_allow_html=True)

class TimetableSystem:
    def __init__(self):
        self.load_data()
        self.days = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY']
        self.time_slots = [('9:00 AM', '10:30 AM'), ('10:45 AM', '12:15 PM'), ('12:30 PM', '2:00 PM'), ('2:15 PM', '3:45 PM')]
        self.variables = []
        self.domains = {}
        
    def load_data(self):
        self.instructors_df = pd.read_csv('Instructors.csv')
        self.rooms_df = pd.read_csv('Rooms.csv')
        self.students_df = pd.read_csv('Students.csv')
        self.timeslots_df = pd.read_csv('TimeSlots.csv')
        self.courses_df = pd.read_csv('Courses.csv')
        self.courses_df = self.courses_df.loc[:, ~self.courses_df.columns.str.contains('^Unnamed')]
        self.timeslots_df = self.timeslots_df[self.timeslots_df['SUB_DURATION'] == '90 Min']
        self.courses_df = self.courses_df[self.courses_df['Course_code'].notna()]
        self.courses_df = self.courses_df[self.courses_df['Type'] != 'DAY']
        
    def find_rooms(self, course_type, capacity):
        if course_type == 'Lec':
            rooms = self.rooms_df[(self.rooms_df['Type'] == 'Lec') & (self.rooms_df['MAX_Capacity'] >= capacity)]['BUILDING_NAME'].tolist()
        elif course_type == 'Lab' or course_type == 'PHY-LAB':
            rooms = self.rooms_df[(self.rooms_df['Type'].isin(['Lab', 'PHY-Lab'])) & (self.rooms_df['MAX_Capacity'] >= capacity)]['BUILDING_NAME'].tolist()
        elif course_type == 'Sec':
            rooms = self.rooms_df[(self.rooms_df['Type'] == 'Sec') & (self.rooms_df['MAX_Capacity'] >= capacity)]['BUILDING_NAME'].tolist()
        else:
            rooms = self.rooms_df['BUILDING_NAME'].tolist()
        return rooms if rooms else ['RED HALL']
    
    def create_model(self):
        for _, course in self.courses_df.iterrows():
            if pd.isna(course['Course_code']):
                continue
                
            year = course['YEAR']
            program = course['Program']
            course_code = course['Course_code']
            course_type = course['Type']
            instructor = course['Instructor']
            
            num_groups = int(course['NO_of_groups']) if not pd.isna(course['NO_of_groups']) else 1
            num_sections = int(course['NO_of_sections']) if not pd.isna(course['NO_of_sections']) else 1
            
            if course_type == 'Lec':
                for group in range(1, num_groups + 1):
                    variable_id = f"{course_code}_LEC_{program}_G{group}"
                    self.variables.append(variable_id)
                    
                    domain = []
                    for day in self.days:
                        for time_idx, (start, end) in enumerate(self.time_slots):
                            suitable_rooms = self.find_rooms(course_type, 80)
                            for room in suitable_rooms:
                                domain.append({
                                    'day': day, 'time_slot': time_idx, 'time_start': start, 'time_end': end,
                                    'room': room, 'instructor': instructor, 'course_code': course_code,
                                    'course_type': course_type, 'program': program, 'group': f"G{group}",
                                    'sections': f"S1-S{num_sections}", 'duration': '90 Min', 'year': year
                                })
                    self.domains[variable_id] = domain
            
            elif course_type in ['Lab', 'Sec', 'PHY-LAB']:
                for group in range(1, num_groups + 1):
                    for section in range(1, num_sections + 1):
                        variable_id = f"{course_code}_{course_type}_{program}_G{group}_S{section}"
                        self.variables.append(variable_id)
                        
                        domain = []
                        for day in self.days:
                            for time_idx, (start, end) in enumerate(self.time_slots):
                                room_capacity = 25 if course_type == 'Sec' else 50
                                suitable_rooms = self.find_rooms(course_type, room_capacity)
                                for room in suitable_rooms:
                                    domain.append({
                                        'day': day, 'time_slot': time_idx, 'time_start': start, 'time_end': end,
                                        'room': room, 'instructor': instructor, 'course_code': course_code,
                                        'course_type': course_type, 'program': program, 'group': f"G{group}",
                                        'section': f"S{section}", 'duration': '90 Min', 'year': year
                                    })
                        self.domains[variable_id] = domain
    
    def generate(self):
        assignment = {}
        used_slots = {}
        instructor_slots = {}
        
        lecture_vars = [v for v in self.variables if 'LEC' in v]
        for var in lecture_vars:
            assigned = False
            for day in self.days:
                for time_idx, (start, end) in enumerate(self.time_slots):
                    if assigned: break
                    if var not in self.domains or not self.domains[var]: continue
                    
                    instructor = self.domains[var][0]['instructor']
                    instructor_key = (instructor, day, time_idx)
                    if instructor_key in instructor_slots: continue
                    
                    suitable_rooms = self.find_rooms('Lec', 80)
                    for room in suitable_rooms:
                        slot_key = (day, time_idx, room)
                        if slot_key not in used_slots:
                            assignment[var] = self.domains[var][0].copy()
                            assignment[var].update({
                                'day': day, 'time_slot': time_idx, 'time_start': start, 
                                'time_end': end, 'room': room
                            })
                            used_slots[slot_key] = True
                            instructor_slots[instructor_key] = True
                            assigned = True
                            break
                if assigned: break
        
        lab_vars = [v for v in self.variables if 'LEC' not in v]
        for var in lab_vars:
            assigned = False
            for day in self.days:
                for time_idx, (start, end) in enumerate(self.time_slots):
                    if assigned: break
                    if var not in self.domains or not self.domains[var]: continue
                    
                    instructor = self.domains[var][0]['instructor']
                    instructor_key = (instructor, day, time_idx)
                    if instructor_key in instructor_slots: continue
                    
                    course_type = var.split('_')[1]
                    room_capacity = 25 if course_type == 'Sec' else 50
                    suitable_rooms = self.find_rooms(course_type, room_capacity)
                    for room in suitable_rooms:
                        slot_key = (day, time_idx, room)
                        if slot_key not in used_slots:
                            assignment[var] = self.domains[var][0].copy()
                            assignment[var].update({
                                'day': day, 'time_slot': time_idx, 'time_start': start, 
                                'time_end': end, 'room': room
                            })
                            used_slots[slot_key] = True
                            instructor_slots[instructor_key] = True
                            assigned = True
                            break
                if assigned: break
        
        return assignment
    
    def show_timetable(self, assignment):
        if not assignment:
            st.error("No timetable")
            return
            
        timetable_data = []
        for var, assign in assignment.items():
            timetable_data.append({
                'Course Code': assign['course_code'],
                'Course Type': assign['course_type'],
                'Day': assign['day'],
                'Time': f"{assign['time_start']} - {assign['time_end']}",
                'Room': assign['room'],
                'Instructor': assign['instructor'],
                'Program': assign['program'],
                'Group': assign['group'],
                'Section': assign.get('section', 'ALL'),
                'Year': assign.get('year', 'N/A')
            })
        
        timetable_df = pd.DataFrame(timetable_data)
        
        col1, col2, col3, col4 = st.columns(4)
        with col1:
            years = ['All'] + sorted(timetable_df['Year'].unique())
            selected_year = st.selectbox('Year:', years)
        with col2:
            programs = ['All'] + sorted(timetable_df['Program'].unique())
            selected_program = st.selectbox('Program:', programs)
        with col3:
            groups = ['All'] + sorted(timetable_df['Group'].unique())
            selected_group = st.selectbox('Group:', groups)
        with col4:
            view_type = st.selectbox('View:', ['By Group', 'By Room', 'By Instructor'])
        
        filtered_df = timetable_df.copy()
        if selected_year != 'All':
            filtered_df = filtered_df[filtered_df['Year'] == selected_year]
        if selected_program != 'All':
            filtered_df = filtered_df[filtered_df['Program'] == selected_program]
        if selected_group != 'All':
            filtered_df = filtered_df[filtered_df['Group'] == selected_group]
        
        if view_type == 'By Group':
            self.show_group_view(filtered_df)
        elif view_type == 'By Room':
            self.show_room_view(filtered_df)
        elif view_type == 'By Instructor':
            self.show_instructor_view(filtered_df)
        
        csv = filtered_df.to_csv(index=False)
        st.download_button(
            label="Download CSV",
            data=csv,
            file_name=f"timetable_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv",
            mime="text/csv"
        )
    
    def show_group_view(self, df):
        st.subheader("Group Timetable")
        groups = sorted(df['Group'].unique())
        
        for group in groups:
            group_df = df[df['Group'] == group]
            
            with st.expander(f"Group {group}", expanded=True):
                lectures_df = group_df[group_df['Section'] == 'ALL']
                labs_df = group_df[group_df['Section'] != 'ALL']
                
                st.write("Lectures (All Sections)")
                if not lectures_df.empty:
                    st.dataframe(lectures_df[['Course Code', 'Day', 'Time', 'Room', 'Instructor']], use_container_width=True)
                
                st.write("Labs & Sections")
                if not labs_df.empty:
                    st.dataframe(labs_df[['Course Code', 'Course Type', 'Day', 'Time', 'Room', 'Instructor', 'Section']], use_container_width=True)
                
                days = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY']
                time_slots = ['9:00 AM - 10:30 AM', '10:45 AM - 12:15 PM', '12:30 PM - 2:00 PM', '2:15 PM - 3:45 PM']
                
                schedule_data = []
                for time_slot in time_slots:
                    row = {'Time Slot': time_slot}
                    for day in days:
                        day_classes = group_df[group_df['Day'] == day]
                        time_class = day_classes[day_classes['Time'] == time_slot]
                        
                        if not time_class.empty:
                            class_info = time_class.iloc[0]
                            if class_info['Section'] == 'ALL':
                                cell_text = f"LEC: {class_info['Course Code']}\n{class_info['Room']}\n{class_info['Instructor']}"
                            else:
                                cell_text = f"{class_info['Course Type']}: {class_info['Course Code']}\n{class_info['Room']}\n{class_info['Instructor']}"
                            row[day] = cell_text
                        else:
                            row[day] = "Free"
                    
                    schedule_data.append(row)
                
                schedule_df = pd.DataFrame(schedule_data)
                st.dataframe(schedule_df, use_container_width=True)
    
    def show_room_view(self, df):
        st.subheader("Room Schedule")
        rooms = sorted(df['Room'].unique())
        
        for room in rooms:
            room_df = df[df['Room'] == room]
            
            with st.expander(f"Room {room}", expanded=False):
                if room_df.empty:
                    st.write("No classes")
                    continue
                    
                days = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY']
                time_slots = ['9:00 AM - 10:30 AM', '10:45 AM - 12:15 PM', '12:30 PM - 2:00 PM', '2:15 PM - 3:45 PM']
                
                schedule_data = []
                for time_slot in time_slots:
                    row = {'Time Slot': time_slot}
                    for day in days:
                        day_classes = room_df[room_df['Day'] == day]
                        time_class = day_classes[day_classes['Time'] == time_slot]
                        
                        if not time_class.empty:
                            class_info = time_class.iloc[0]
                            if class_info['Section'] == 'ALL':
                                cell_text = f"LEC: {class_info['Course Code']}\n{class_info['Group']}\n{class_info['Instructor']}"
                            else:
                                cell_text = f"{class_info['Course Type']}: {class_info['Course Code']}\n{class_info['Group']} {class_info['Section']}\n{class_info['Instructor']}"
                            row[day] = cell_text
                        else:
                            row[day] = "Available"
                    
                    schedule_data.append(row)
                
                schedule_df = pd.DataFrame(schedule_data)
                st.dataframe(schedule_df, use_container_width=True)
    
    def show_instructor_view(self, df):
        st.subheader("Instructor Schedule")
        instructors = sorted(df['Instructor'].unique())
        
        for instructor in instructors:
            instructor_df = df[df['Instructor'] == instructor]
            
            with st.expander(f"{instructor}", expanded=False):
                if instructor_df.empty:
                    st.write("No classes")
                    continue
                    
                days = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY']
                time_slots = ['9:00 AM - 10:30 AM', '10:45 AM - 12:15 PM', '12:30 PM - 2:00 PM', '2:15 PM - 3:45 PM']
                
                schedule_data = []
                for time_slot in time_slots:
                    row = {'Time Slot': time_slot}
                    for day in days:
                        day_classes = instructor_df[instructor_df['Day'] == day]
                        time_class = day_classes[day_classes['Time'] == time_slot]
                        
                        if not time_class.empty:
                            class_info = time_class.iloc[0]
                            if class_info['Section'] == 'ALL':
                                cell_text = f"LEC: {class_info['Course Code']}\n{class_info['Room']}\n{class_info['Group']}"
                            else:
                                cell_text = f"{class_info['Course Type']}: {class_info['Course Code']}\n{class_info['Room']}\n{class_info['Group']} {class_info['Section']}"
                            row[day] = cell_text
                        else:
                            row[day] = "Free"
                    
                    schedule_data.append(row)
                
                schedule_df = pd.DataFrame(schedule_data)
                st.dataframe(schedule_df, use_container_width=True)

def main():
    st.markdown("<h1 style='text-align: center; color: black;'>Timetable System</h1>", unsafe_allow_html=True)
    
    if 'system' not in st.session_state:
        st.session_state.system = TimetableSystem()
        st.session_state.timetable = None
    
    st.sidebar.header("Controls")
    
    if st.sidebar.button("Generate Timetable", type="primary"):
        with st.spinner("Generating..."):
            st.session_state.system.create_model()
            st.session_state.timetable = st.session_state.system.generate()
    
    if st.session_state.timetable:
        st.session_state.system.show_timetable(st.session_state.timetable)
    else:
        st.write("Generate timetable to begin")

if __name__ == "__main__":
    main()
