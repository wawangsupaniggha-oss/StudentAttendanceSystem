using System;
using System.Data;
using System.Data.SqlClient;
using System.Web.UI;

namespace StudentAttendanceSystem
{
    public partial class Default : Page
    {
        string conStr = @"Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=AttendanceDB;Integrated Security=True";

        protected void Page_Load(object sender, EventArgs e)
        {
            if (!IsPostBack)
            {
                LoadStudents();
                LoadAttendance();
            }
        }

        void LoadStudents()
        {
            using (SqlConnection con = new SqlConnection(conStr))
            {
                SqlDataAdapter da = new SqlDataAdapter("SELECT * FROM Students", con);
                DataTable dt = new DataTable();
                da.Fill(dt);

                ddlStudents.DataSource = dt;
                ddlStudents.DataTextField = "Name";
                ddlStudents.DataValueField = "Id";
                ddlStudents.DataBind();
            }
        }

        void LoadAttendance()
        {
            using (SqlConnection con = new SqlConnection(conStr))
            {
                SqlDataAdapter da = new SqlDataAdapter(@"
                SELECT A.Id, S.Name, A.Subject, A.Date, A.TimeIn, A.Status, A.HoursLate
                FROM Attendance A
                JOIN Students S ON A.StudentId = S.Id", con);

                DataTable dt = new DataTable();
                da.Fill(dt);

                GridView1.DataSource = dt;
                GridView1.DataBind();
            }
        }

        protected void btnAddStudent_Click(object sender, EventArgs e)
        {
            using (SqlConnection con = new SqlConnection(conStr))
            {
                string query = "INSERT INTO Students (Name, Email, Course) VALUES (@n, @e, @c)";
                SqlCommand cmd = new SqlCommand(query, con);

                cmd.Parameters.AddWithValue("@n", txtName.Text);
                cmd.Parameters.AddWithValue("@e", txtEmail.Text);
                cmd.Parameters.AddWithValue("@c", txtCourse.Text);

                con.Open();
                cmd.ExecuteNonQuery();
            }

            LoadStudents();
        }

        protected void btnSearch_Click(object sender, EventArgs e)
        {
            using (SqlConnection con = new SqlConnection(conStr))
            {
                string query = @"
                SELECT A.Id, S.Name, A.Subject, A.Date, A.TimeIn, A.Status, A.HoursLate
                FROM Attendance A
                JOIN Students S ON A.StudentId = S.Id
                WHERE S.Name LIKE @name";

                SqlDataAdapter da = new SqlDataAdapter(query, con);
                da.SelectCommand.Parameters.AddWithValue("@name", "%" + txtSearch.Text + "%");

                DataTable dt = new DataTable();
                da.Fill(dt);

                GridView1.DataSource = dt;
                GridView1.DataBind();
            }
        }

        protected void btnMark_Click(object sender, EventArgs e)
        {
            int studentId = int.Parse(ddlStudents.SelectedValue);
            DateTime now = DateTime.Now;

            string subject = GetSubject(now);
            DateTime classStart = GetClassStart(now);

            string status = "Present";
            double hoursLate = 0;

            using (SqlConnection con = new SqlConnection(conStr))
            {
                // PREVENT DUPLICATE
                string check = @"SELECT COUNT(*) FROM Attendance 
                WHERE StudentId=@sid AND Date=@date AND Subject=@sub";

                SqlCommand checkCmd = new SqlCommand(check, con);
                checkCmd.Parameters.AddWithValue("@sid", studentId);
                checkCmd.Parameters.AddWithValue("@date", now.Date);
                checkCmd.Parameters.AddWithValue("@sub", subject);

                con.Open();
                int count = (int)checkCmd.ExecuteScalar();

                if (count > 0)
                {
                    Response.Write("<script>alert('Already recorded!');</script>");
                    return;
                }

                if (subject == "No Class")
                    status = "No Class";
                else if (now > classStart)
                {
                    status = "Late";
                    hoursLate = Math.Round((now - classStart).TotalHours, 2);
                }

                string query = @"INSERT INTO Attendance 
                (StudentId, Subject, Date, TimeIn, Status, HoursLate)
                VALUES (@sid, @sub, @date, @time, @status, @hours)";

                SqlCommand cmd = new SqlCommand(query, con);

                cmd.Parameters.AddWithValue("@sid", studentId);
                cmd.Parameters.AddWithValue("@sub", subject);
                cmd.Parameters.AddWithValue("@date", now.Date);
                cmd.Parameters.AddWithValue("@time", now.TimeOfDay);
                cmd.Parameters.AddWithValue("@status", status);
                cmd.Parameters.AddWithValue("@hours", hoursLate);

                cmd.ExecuteNonQuery();
            }

            LoadAttendance();
        }

        string GetSubject(DateTime now)
        {
            var d = now.DayOfWeek;
            var t = now.TimeOfDay;

            if (d == DayOfWeek.Monday || d == DayOfWeek.Wednesday)
            {
                if (t >= new TimeSpan(10, 30, 0) && t < new TimeSpan(11, 30, 0)) return "Environmental Science";
                if (t >= new TimeSpan(11, 30, 0) && t < new TimeSpan(12, 30, 0)) return "Masining";
                if (t >= new TimeSpan(13, 0, 0) && t < new TimeSpan(14, 0, 0)) return "Philippine History";
            }

            if (d == DayOfWeek.Monday && t >= new TimeSpan(17, 0, 0))
                return "Pathfit";

            if (d == DayOfWeek.Tuesday || d == DayOfWeek.Thursday)
            {
                if (t >= new TimeSpan(9, 0, 0) && t < new TimeSpan(13, 0, 0)) return "Web System";
                if (t >= new TimeSpan(10, 0, 0) && t < new TimeSpan(11, 0, 0)) return "Discrete Math";
                if (t >= new TimeSpan(13, 0, 0) && t < new TimeSpan(14, 0, 0)) return "Quantitative Methods";
            }

            if (d == DayOfWeek.Friday && t >= new TimeSpan(12, 0, 0))
                return "Programming 2";

            return "No Class";
        }

        DateTime GetClassStart(DateTime now)
        {
            var d = now.DayOfWeek;
            var t = now.TimeOfDay;

            if (d == DayOfWeek.Monday || d == DayOfWeek.Wednesday)
            {
                if (t >= new TimeSpan(10, 30, 0) && t < new TimeSpan(11, 30, 0)) return now.Date.AddHours(10.5);
                if (t >= new TimeSpan(11, 30, 0) && t < new TimeSpan(12, 30, 0)) return now.Date.AddHours(11.5);
                if (t >= new TimeSpan(13, 0, 0) && t < new TimeSpan(14, 0, 0)) return now.Date.AddHours(13);
            }

            if (d == DayOfWeek.Monday && t >= new TimeSpan(17, 0, 0))
                return now.Date.AddHours(17);

            if (d == DayOfWeek.Tuesday || d == DayOfWeek.Thursday)
            {
                if (t >= new TimeSpan(9, 0, 0) && t < new TimeSpan(13, 0, 0)) return now.Date.AddHours(9);
                if (t >= new TimeSpan(10, 0, 0) && t < new TimeSpan(11, 0, 0)) return now.Date.AddHours(10);
                if (t >= new TimeSpan(13, 0, 0) && t < new TimeSpan(14, 0, 0)) return now.Date.AddHours(13);
            }

            if (d == DayOfWeek.Friday && t >= new TimeSpan(12, 0, 0))
                return now.Date.AddHours(12);

            return now;
        }
    }
}
