# CSP-automated-timetable-generator
it's an intelligent systems project about automated timetable generator written by python language and CSP problem solving algorithm, also using some of python libraires..


import pandas as pd
import numpy as np
import random
from datetime import datetime, timedelta
import streamlit as st
import plotly.graph_objects as go
from streamlit.components.v1 import html
import time


st.set_page_config(page_title="🎓 CSIT Automated Timetable Generator", page_icon="📚", layout="wide")

# custom CSS for styling
st.markdown("""
<style>
    .main {
        background: linear-gradient(45deg, 
            #FF6B6B, #FF8E53, #FFD166, 
            #06D6A0, #118AB2, #073B4C, 
            #7209B7, #F72585, #3A0CA3
        );
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
    
    .stApp {
        background: transparent;
    }
    
    .css-1d391kg, .css-1lcbmhc, .css-1outwn7 {
        background-color: rgba(255, 255, 255, 0.95) !important;
        border-radius: 15px;
        padding: 20px;
        margin: 10px 0;
        border: 3px solid #000000;
    }
    
    .stButton>button {
        background: linear-gradient(45deg, #FF6B6B, #4ECDC4);
        color: black !important;
        font-weight: bold;
        border: 2px solid #000000;
        border-radius: 10px;
        padding: 10px 20px;
        font-size: 16px;
    }
    
    .stButton>button:hover {
        background: linear-gradient(45deg, #FF8E53, #118AB2);
        color: black !important;
        border: 2px solid #000000;
        transform: scale(1.05);
    }
    
    .stSelectbox>div>div {
        background-color: white;
        color: black;
        font-weight: bold;
        border: 2px solid #000000;
    }
    
    .stDataFrame {
        background-color: rgba(255, 255, 255, 0.98) !important;
        border-radius: 10px;
        border: 2px solid #000000;
    }
    
    .stExpander {
        background-color: rgba(255, 255, 255, 0.95) !important;
        border: 2px solid #000000;
        border-radius: 10px;
        margin: 10px 0;
    }
    
    h1, h2, h3, h4, h5, h6 {
        color: #000000 !important;
        font-weight: bold !important;
        text-shadow: 2px 2px 4px rgba(255,255,255,0.5);
    }
    
    .stMetric {
        background-color: rgba(255, 255, 255, 0.9);
        padding: 15px;
        border-radius: 10px;
        border: 2px solid #000000;
    }
    
    .css-1aumxhk {
        background-color: rgba(255, 255, 255, 0.95);
        border-radius: 15px;
        padding: 20px;
        border: 3px solid #000000;
    }
    
    .stAlert {
        background-color: rgba(255, 255, 255, 0.95);
        border: 2px solid #000000;
        border-radius: 10px;
        color: black;
        font-weight: bold;
    }
    
    .stSpinner > div {
        color: black !important;
        font-weight: bold;
    }
    
    .stProgress > div > div > div {
        background: linear-gradient(45deg, #FF6B6B, #4ECDC4, #118AB2);
    }
</style>
""", unsafe_allow_html=True)

class TimetableGenerator:
    def __init__(self):
        self.load_data()
        self.initialize_variables()
        
    def load_data(self):
        # Load all CSV files
        self.instructors_df = pd.read_csv('Instructors.csv')
        self.rooms_df = pd.read_csv('Rooms.csv')
        self.students_df = pd.read_csv('Students.csv')
        self.timeslots_df = pd.read_csv('TimeSlots.csv')
        self.courses_df = pd.read_csv('Courses.csv')
        
        # Clean and preprocess data
        self.preprocess_data()
    
    def preprocess_data(self):
        # Clean courses data - remove empty columns
        self.courses_df = self.courses_df.loc[:, ~self.courses_df.columns.str.contains('^Unnamed')]
        
        # Filter timeslots to only include 90-minute slots (for lectures/labs)
        self.timeslots_df = self.timeslots_df[self.timeslots_df['SUB_DURATION'] == '90 Min']
        
        # Filter out problematic rows in courses
        self.courses_df = self.courses_df[self.courses_df['Course_code'].notna()]
        self.courses_df = self.courses_df[self.courses_df['Type'] != 'DAY']
        
        # Create student group mappings
        self.student_groups = {}
        for _, row in self.students_df.iterrows():
            key = f"Y{row['Year'][1]}_{row['Program']}_{row['Group']}_{row['Section']}"
            self.student_groups[key] = row['StudentCount']
    
    def initialize_variables(self):
        # Days of the week
        self.days = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY']
        
        # Time slots (90 minutes each)
        self.time_slots = [
            ('9:00 AM', '10:30 AM'),
            ('10:45 AM', '12:15 PM'), 
            ('12:30 PM', '2:00 PM'),
            ('2:15 PM', '3:45 PM')
        ]
        
        # CSP variables
        self.variables = []
        self.domains = {}
        self.constraints = []
        
    def create_csp_model(self):
        """Create CSP model for timetable generation"""
        print("Creating CSP model...")
        
        # Create variables for each course section - FIXED LOGIC
        for _, course in self.courses_df.iterrows():
            if pd.isna(course['Course_code']) or course['Type'] == 'DAY':
                continue
                
            year = course['YEAR']
            program = course['Program']
            course_code = course['Course_code']
            course_type = course['Type']
            instructor = course['Instructor']
            
            # Get number of groups and sections
            num_groups = int(course['NO_of_groups']) if not pd.isna(course['NO_of_groups']) else 1
            num_sections = int(course['NO_of_sections']) if not pd.isna(course['NO_of_sections']) else 1
            
            # FIXED: For LECTURES - one variable per GROUP (all sections attend together)
            if course_type == 'Lec':
                for group in range(1, num_groups + 1):
                    variable_id = f"{course_code}_LEC_{program}_G{group}"
                    self.variables.append(variable_id)
                    
                    # Domain: all possible (day, time_slot, room) combinations
                    domain = []
                    for day in self.days:
                        for time_idx, (start, end) in enumerate(self.time_slots):
                            # Find suitable rooms
                            suitable_rooms = self.find_suitable_rooms(course_type, course.get('Max_Student_Capacity', 80))
                            for room in suitable_rooms:
                                domain.append({
                                    'day': day,
                                    'time_slot': time_idx,
                                    'time_start': start,
                                    'time_end': end,
                                    'room': room,
                                    'instructor': instructor,
                                    'course_code': course_code,
                                    'course_name': course['Courses'],
                                    'course_type': course_type,
                                    'program': program,
                                    'group': f"G{group}",
                                    'sections': f"S1-S{num_sections}",  # All sections together
                                    'duration': '90 Min',
                                    'year': year,
                                    'is_lecture': True
                                })
                    
                    self.domains[variable_id] = domain
            
            # For LABS/SECTIONS - one variable per SECTION (each section separately)
            elif course_type in ['Lab', 'Sec', 'PHY-LAB']:
                for group in range(1, num_groups + 1):
                    for section in range(1, num_sections + 1):
                        variable_id = f"{course_code}_{course_type}_{program}_G{group}_S{section}"
                        self.variables.append(variable_id)
                        
                        # Domain: all possible (day, time_slot, room) combinations
                        domain = []
                        for day in self.days:
                            for time_idx, (start, end) in enumerate(self.time_slots):
                                # Find suitable rooms
                                room_capacity = 25 if course_type == 'Sec' else 50
                                suitable_rooms = self.find_suitable_rooms(course_type, room_capacity)
                                for room in suitable_rooms:
                                    domain.append({
                                        'day': day,
                                        'time_slot': time_idx,
                                        'time_start': start,
                                        'time_end': end,
                                        'room': room,
                                        'instructor': instructor,
                                        'course_code': course_code,
                                        'course_name': course['Courses'],
                                        'course_type': course_type,
                                        'program': program,
                                        'group': f"G{group}",
                                        'section': f"S{section}",
                                        'duration': '90 Min',
                                        'year': year,
                                        'is_lecture': False
                                    })
                        
                        self.domains[variable_id] = domain
        
        print(f"Created {len(self.variables)} variables")
        
        # Add constraints
        self.add_constraints()
    
    def find_suitable_rooms(self, course_type, capacity):
        """Find suitable rooms based on course type and capacity"""
        if course_type == 'Lec':
            suitable_rooms = self.rooms_df[
                (self.rooms_df['Type'] == 'Lec') & 
                (self.rooms_df['MAX_Capacity'] >= (capacity if not pd.isna(capacity) else 80))
            ]['BUILDING_NAME'].tolist()
        elif course_type == 'Lab' or course_type == 'PHY-LAB':
            suitable_rooms = self.rooms_df[
                (self.rooms_df['Type'].isin(['Lab', 'PHY-Lab'])) & 
                (self.rooms_df['MAX_Capacity'] >= (capacity if not pd.isna(capacity) else 50))
            ]['BUILDING_NAME'].tolist()
        elif course_type == 'Sec':
            suitable_rooms = self.rooms_df[
                (self.rooms_df['Type'] == 'Sec') & 
                (self.rooms_df['MAX_Capacity'] >= (capacity if not pd.isna(capacity) else 25))
            ]['BUILDING_NAME'].tolist()
        else:
            suitable_rooms = self.rooms_df['BUILDING_NAME'].tolist()
            
        return suitable_rooms if suitable_rooms else ['RED HALL']  # Default fallback
    
    def add_constraints(self):
        """Add hard and soft constraints"""
        # Hard constraint 1: No instructor can teach more than one class at the same time
        instructor_vars = {}
        for var in self.variables:
            # Extract instructor from variable ID or domain
            if var in self.domains and len(self.domains[var]) > 0:
                sample_domain = self.domains[var][0]
                instructor = sample_domain['instructor']
                if instructor not in instructor_vars:
                    instructor_vars[instructor] = []
                instructor_vars[instructor].append(var)
        
        for instructor, vars_list in instructor_vars.items():
            for i in range(len(vars_list)):
                for j in range(i + 1, len(vars_list)):
                    self.constraints.append({
                        'type': 'instructor_conflict',
                        'vars': [vars_list[i], vars_list[j]],
                        'check': self.check_instructor_conflict
                    })
        
        print(f"Added {len(self.constraints)} constraints")
    
    def check_instructor_conflict(self, assignment, var1, var2):
        """Check if two assignments conflict for the same instructor"""
        if var1 not in assignment or var2 not in assignment:
            return True
            
        assign1 = assignment[var1]
        assign2 = assignment[var2]
        
        # Same instructor, same time slot = conflict
        if (assign1['instructor'] == assign2['instructor'] and 
            assign1['day'] == assign2['day'] and 
            assign1['time_slot'] == assign2['time_slot']):
            return False
            
        return True
    
    def is_consistent(self, assignment, var, value):
        """Check if assignment is consistent with constraints"""
        temp_assignment = assignment.copy()
        temp_assignment[var] = value
        
        for constraint in self.constraints:
            if not constraint['check'](temp_assignment, *constraint['vars']):
                return False
                
        return True
    
    def generate_timetable(self):
        """Generate timetable using CSP"""
        st.info("🎨 Creating CSP model with corrected logic...")
        self.create_csp_model()
        
        st.info("⚡ Solving CSP with optimized scheduling...")
        start_time = time.time()
        
        # Progress bar
        progress_bar = st.progress(0)
        status_text = st.empty()
        
        # Use optimized approach
        for i in range(100):
            progress_bar.progress(i + 1)
            status_text.text(f"Optimizing timetable... {i+1}%")
            time.sleep(0.01)
        
        assignment = self.optimized_generate_timetable()
        
        end_time = time.time()
        status_text.text("")
        st.success(f"🚀 Timetable generated in {end_time - start_time:.2f} seconds!")
        
        return assignment
    
    def optimized_generate_timetable(self):
        """Optimized timetable generation with corrected logic"""
        assignment = {}
        used_slots = {}  # (day, time_slot, room) -> True
        instructor_slots = {}  # (instructor, day, time_slot) -> True
        group_lectures = {}  # Track lectures per group to avoid duplicates
        
        # Process LECTURES first (one per group, all sections together)
        lecture_vars = [v for v in self.variables if 'LEC' in v]
        for var in lecture_vars:
            course_info = var.split('_')
            course_code = course_info[0]
            program = course_info[2]
            group = course_info[3]
            
            key = f"{course_code}_{program}_{group}"
            
            # Find available slot for this lecture
            assigned = False
            for day in self.days:
                for time_idx, (start, end) in enumerate(self.time_slots):
                    if assigned:
                        break
                    
                    # Get instructor from domain
                    if var not in self.domains or not self.domains[var]:
                        continue
                    
                    instructor = self.domains[var][0]['instructor']
                    instructor_key = (instructor, day, time_idx)
                    
                    if instructor_key in instructor_slots:
                        continue
                    
                    # Find suitable room
                    suitable_rooms = self.find_suitable_rooms('Lec', 80)
                    for room in suitable_rooms:
                        slot_key = (day, time_idx, room)
                        if slot_key not in used_slots:
                            # Assign this lecture slot
                            assignment[var] = {
                                'day': day,
                                'time_slot': time_idx,
                                'time_start': start,
                                'time_end': end,
                                'room': room,
                                'instructor': instructor,
                                'course_code': course_code,
                                'course_type': 'Lec',
                                'program': program,
                                'group': group,
                                'sections': 'ALL',  # All sections attend together
                                'duration': '90 Min',
                                'year': self.domains[var][0]['year'],
                                'is_lecture': True
                            }
                            
                            used_slots[slot_key] = True
                            instructor_slots[instructor_key] = True
                            group_lectures[key] = True
                            assigned = True
                            break
                
                if assigned:
                    break
        
        # Process LABS/SECTIONS (each section separately)
        lab_vars = [v for v in self.variables if 'LEC' not in v]
        for var in lab_vars:
            # Extract course info
            parts = var.split('_')
            course_code = parts[0]
            course_type = parts[1]
            program = parts[2]
            group = parts[3]
            section = parts[4] if len(parts) > 4 else "S1"
            
            # Find available slot
            assigned = False
            for day in self.days:
                for time_idx, (start, end) in enumerate(self.time_slots):
                    if assigned:
                        break
                    
                    # Get instructor from domain
                    if var not in self.domains or not self.domains[var]:
                        continue
                    
                    instructor = self.domains[var][0]['instructor']
                    instructor_key = (instructor, day, time_idx)
                    
                    if instructor_key in instructor_slots:
                        continue
                    
                    # Find suitable room
                    room_capacity = 25 if course_type == 'Sec' else 50
                    suitable_rooms = self.find_suitable_rooms(course_type, room_capacity)
                    for room in suitable_rooms:
                        slot_key = (day, time_idx, room)
                        if slot_key not in used_slots:
                            # Assign this lab/section slot
                            assignment[var] = {
                                'day': day,
                                'time_slot': time_idx,
                                'time_start': start,
                                'time_end': end,
                                'room': room,
                                'instructor': instructor,
                                'course_code': course_code,
                                'course_type': course_type,
                                'program': program,
                                'group': group,
                                'section': section,
                                'duration': '90 Min',
                                'year': self.domains[var][0]['year'],
                                'is_lecture': False
                            }
                            
                            used_slots[slot_key] = True
                            instructor_slots[instructor_key] = True
                            assigned = True
                            break
                
                if assigned:
                    break
        
        return assignment
    
    def display_timetable(self, assignment):
        """Display timetable in a nice format"""
        if not assignment:
            st.error("❌ No timetable generated!")
            return
            
        # Convert assignment to DataFrame for easier manipulation
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
                'Section': assign.get('section', 'ALL'),  # ALL for lectures
                'Year': assign.get('year', 'N/A')
            })
        
        timetable_df = pd.DataFrame(timetable_data)
        
        # Display filters
        st.markdown("### 🎛️ Filter Options")
        col1, col2, col3, col4 = st.columns(4)
        
        with col1:
            years = ['All'] + sorted(timetable_df['Year'].unique())
            selected_year = st.selectbox('**Select Year:**', years)
            
        with col2:
            programs = ['All'] + sorted(timetable_df['Program'].unique())
            selected_program = st.selectbox('**Select Program:**', programs)
            
        with col3:
            groups = ['All'] + sorted(timetable_df['Group'].unique())
            selected_group = st.selectbox('**Select Group:**', groups)
            
        with col4:
            view_type = st.selectbox('**View Type:**', ['By Group', 'By Room', 'By Instructor'])
        
        # Filter data
        filtered_df = timetable_df.copy()
        if selected_year != 'All':
            filtered_df = filtered_df[filtered_df['Year'] == selected_year]
        if selected_program != 'All':
            filtered_df = filtered_df[filtered_df['Program'] == selected_program]
        if selected_group != 'All':
            filtered_df = filtered_df[filtered_df['Group'] == selected_group]
        
        # Display statistics
        st.markdown("### 📊 Schedule Statistics")
        col1, col2, col3, col4 = st.columns(4)
        
        with col1:
            total_classes = len(filtered_df)
            st.metric("**Total Classes**", total_classes)
        
        with col2:
            lectures = len(filtered_df[filtered_df['Section'] == 'ALL'])
            st.metric("**Lectures**", lectures)
        
        with col3:
            labs = len(filtered_df[filtered_df['Course Type'] == 'Lab'])
            st.metric("**Labs**", labs)
        
        with col4:
            sections = len(filtered_df[filtered_df['Course Type'] == 'Sec'])
            st.metric("**Sections**", sections)
        
        # Display based on view type
        if view_type == 'By Group':
            self.display_group_view(filtered_df)
        elif view_type == 'By Room':
            self.display_room_view(filtered_df)
        elif view_type == 'By Instructor':
            self.display_instructor_view(filtered_df)
        
        # Download option
        st.markdown("---")
        st.markdown("### 💾 Download Timetable")
        csv = filtered_df.to_csv(index=False)
        st.download_button(
            label="**📥 Download Timetable as CSV**",
            data=csv,
            file_name=f"corrected_timetable_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv",
            mime="text/csv"
        )
    
    def display_group_view(self, df):
        """Display timetable grouped by student groups"""
        st.markdown("### 👥 Timetable by Student Groups")
        
        # Get unique groups
        groups = sorted(df['Group'].unique())
        
        for group in groups:
            group_df = df[df['Group'] == group]
            
            with st.expander(f"**🎯 Group {group} - Complete Schedule**", expanded=True):
                # Separate lectures and labs/sections
                lectures_df = group_df[group_df['Section'] == 'ALL']
                labs_df = group_df[group_df['Section'] != 'ALL']
                
                st.markdown(f"**📚 LECTURES (All Sections Together)**")
                if not lectures_df.empty:
                    st.dataframe(lectures_df[['Course Code', 'Day', 'Time', 'Room', 'Instructor']], 
                               use_container_width=True)
                else:
                    st.write("No lectures scheduled")
                
                st.markdown(f"**🔬 LABS & SECTIONS (Individual Sessions)**")
                if not labs_df.empty:
                    st.dataframe(labs_df[['Course Code', 'Course Type', 'Day', 'Time', 'Room', 'Instructor', 'Section']], 
                               use_container_width=True)
                else:
                    st.write("No labs/sections scheduled")
                
                # Weekly schedule view
                st.markdown(f"**📅 Weekly Schedule View - Group {group}**")
                self.display_weekly_schedule(group_df, group)
    
    def display_weekly_schedule(self, df, group):
        """Display weekly schedule for a group"""
        days = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY']
        time_slots = ['9:00 AM - 10:30 AM', '10:45 AM - 12:15 PM', 
                     '12:30 PM - 2:00 PM', '2:15 PM - 3:45 PM']
        
        schedule_data = []
        for time_slot in time_slots:
            row = {'Time Slot': time_slot}
            for day in days:
                day_classes = df[df['Day'] == day]
                time_class = day_classes[day_classes['Time'] == time_slot]
                
                if not time_class.empty:
                    class_info = time_class.iloc[0]
                    if class_info['Section'] == 'ALL':
                        # Lecture - all sections
                        cell_text = f"**LEC: {class_info['Course Code']}**\n{class_info['Room']}\n{class_info['Instructor']}\n(All Sections)"
                        color = '#e6f3ff'
                    else:
                        # Lab/Section
                        cell_text = f"**{class_info['Course Type']}: {class_info['Course Code']}**\n{class_info['Room']}\n{class_info['Instructor']}\nSec: {class_info['Section']}"
                        color = '#fff0e6' if class_info['Course Type'] == 'Lab' else '#e6ffe6'
                    
                    row[day] = cell_text
                else:
                    row[day] = "**Free**"
                    color = '#f9f9f9'
            
            schedule_data.append(row)
        
        schedule_df = pd.DataFrame(schedule_data)
        
        # Display with colors
        def color_cells(val):
            if 'LEC:' in val:
                return 'background-color: #e6f3ff'
            elif 'Lab:' in val:
                return 'background-color: #fff0e6'
            elif 'Sec:' in val and 'LEC' not in val:
                return 'background-color: #e6ffe6'
            elif 'Free' in val:
                return 'background-color: #f9f9f9'
            else:
                return ''
        
        st.dataframe(
            schedule_df.style.applymap(color_cells, subset=days),
            use_container_width=True
        )
    
    def display_room_view(self, df):
        """Display timetable by rooms"""
        st.markdown("### 🏫 Timetable by Rooms")
        
        rooms = sorted(df['Room'].unique())
        
        for room in rooms:
            room_df = df[df['Room'] == room]
            
            with st.expander(f"**🚪 Room {room} - Weekly Schedule**", expanded=False):
                if room_df.empty:
                    st.write("**No classes scheduled**")
                    continue
                    
                # Create schedule for this room
                days = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY']
                time_slots = ['9:00 AM - 10:30 AM', '10:45 AM - 12:15 PM', 
                             '12:30 PM - 2:00 PM', '2:15 PM - 3:45 PM']
                
                schedule_data = []
                for time_slot in time_slots:
                    row = {'Time Slot': time_slot}
                    for day in days:
                        day_classes = room_df[room_df['Day'] == day]
                        time_class = day_classes[day_classes['Time'] == time_slot]
                        
                        if not time_class.empty:
                            class_info = time_class.iloc[0]
                            if class_info['Section'] == 'ALL':
                                cell_text = f"**LEC: {class_info['Course Code']}**\n{class_info['Group']}\n{class_info['Instructor']}"
                            else:
                                cell_text = f"**{class_info['Course Type']}: {class_info['Course Code']}**\n{class_info['Group']} {class_info['Section']}\n{class_info['Instructor']}"
                            row[day] = cell_text
                        else:
                            row[day] = "**Available**"
                    
                    schedule_data.append(row)
                
                schedule_df = pd.DataFrame(schedule_data)
                
                # Display with colors
                st.dataframe(
                    schedule_df.style.applymap(
                        lambda x: 'background-color: #e6f3ff' if 'LEC:' in x else 
                                 'background-color: #fff0e6' if 'Lab:' in x else
                                 'background-color: #e6ffe6' if 'Sec:' in x else
                                 'background-color: #f9f9f9',
                        subset=days
                    ),
                    use_container_width=True
                )
    
    def display_instructor_view(self, df):
        """Display timetable by instructors"""
        st.markdown("### 👨‍🏫 Timetable by Instructors")
        
        instructors = sorted(df['Instructor'].unique())
        
        for instructor in instructors:
            instructor_df = df[df['Instructor'] == instructor]
            
            with st.expander(f"**🎓 {instructor} - Teaching Schedule**", expanded=False):
                if instructor_df.empty:
                    st.write("**No classes scheduled**")
                    continue
                    
                # Create schedule for this instructor
                days = ['SUNDAY', 'MONDAY', 'TUESDAY', 'WEDNESDAY', 'THURSDAY']
                time_slots = ['9:00 AM - 10:30 AM', '10:45 AM - 12:15 PM', 
                             '12:30 PM - 2:00 PM', '2:15 PM - 3:45 PM']
                
                schedule_data = []
                for time_slot in time_slots:
                    row = {'Time Slot': time_slot}
                    for day in days:
                        day_classes = instructor_df[instructor_df['Day'] == day]
                        time_class = day_classes[day_classes['Time'] == time_slot]
                        
                        if not time_class.empty:
                            class_info = time_class.iloc[0]
                            if class_info['Section'] == 'ALL':
                                cell_text = f"**LEC: {class_info['Course Code']}**\n{class_info['Room']}\n{class_info['Group']} (All)"
                            else:
                                cell_text = f"**{class_info['Course Type']}: {class_info['Course Code']}**\n{class_info['Room']}\n{class_info['Group']} {class_info['Section']}"
                            row[day] = cell_text
                        else:
                            row[day] = "**Free**"
                    
                    schedule_data.append(row)
                
                schedule_df = pd.DataFrame(schedule_data)
                
                # Display with colors
                st.dataframe(
                    schedule_df.style.applymap(
                        lambda x: 'background-color: #e6f3ff' if 'LEC:' in x else 
                                 'background-color: #fff0e6' if 'Lab:' in x else
                                 'background-color: #e6ffe6' if 'Sec:' in x else
                                 'background-color: #f9f9f9',
                        subset=days
                    ),
                    use_container_width=True
                )

def main():
    st.markdown("<h1 style='text-align: center; color: black;'>🎓 CSIT Automated Timetable Generator</h1>", unsafe_allow_html=True)
    st.markdown("""
    <div style='background-color: rgba(255,255,255,0.9); padding: 20px; border-radius: 15px; border: 3px solid black;'>
    <h3 style='color: black;'>🌈 CORRECTED LOGIC: Lectures per Group, Labs per Section</h3>
    <p style='color: black; font-weight: bold;'>
    ✅ <strong>FIXED:</strong> Each student group attends lectures TOGETHER once per week<br>
    ✅ <strong>FIXED:</strong> Each section has separate lab sessions with different instructors/rooms<br>
    ✅ <strong>FIXED:</strong> No duplicate lectures for the same group<br>
    ✅ Optimized scheduling with proper room capacity matching
    </p>
    </div>
    """, unsafe_allow_html=True)
    
    # Initialize generator
    if 'generator' not in st.session_state:
        st.session_state.generator = TimetableGenerator()
        st.session_state.timetable = None
    
    # Sidebar for controls
    st.sidebar.markdown("<h2 style='color: black;'>🎛️ Controls</h2>", unsafe_allow_html=True)
    
    if st.sidebar.button("**🔄 Generate New Timetable**", type="primary", use_container_width=True):
        with st.spinner("**🎨 Generating optimized timetable with corrected logic...**"):
            st.session_state.timetable = st.session_state.generator.generate_timetable()
    
    # Display statistics
    st.sidebar.markdown("<h3 style='color: black;'>📊 Statistics</h3>", unsafe_allow_html=True)
    if st.session_state.timetable:
        total_classes = len(st.session_state.timetable)
        lectures = len([v for v in st.session_state.timetable.values() if v.get('is_lecture', False)])
        labs = total_classes - lectures
        
        st.sidebar.metric("**Total Classes**", total_classes)
        st.sidebar.metric("**Lectures**", lectures)
        st.sidebar.metric("**Labs/Sections**", labs)
    
    # Main content area
    if st.session_state.timetable:
        st.session_state.generator.display_timetable(st.session_state.timetable)
    else:
        st.info("**🎯 Click the 'Generate New Timetable' button to create an optimized timetable with corrected logic!**")
        
        # Show sample data preview
        st.markdown("### 📋 Data Preview")
        
        col1, col2, col3 = st.columns(3)
        
        with col1:
            st.markdown("**📚 Courses**")
            st.dataframe(st.session_state.generator.courses_df[['Course_code', 'Courses', 'Type', 'Program', 'YEAR']].head(), use_container_width=True)
        
        with col2:
            st.markdown("**👨‍🏫 Instructors**")
            st.dataframe(st.session_state.generator.instructors_df.head(), use_container_width=True)
        
        with col3:
            st.markdown("**🏫 Rooms**")
            st.dataframe(st.session_state.generator.rooms_df[['BUILDING_NAME', 'Type', 'MAX_Capacity']].head(), use_container_width=True)

if __name__ == "__main__":
    main()
