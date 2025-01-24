# Library-Database-Management-System
![Strategic_Spotlighht_large-02](https://github.com/user-attachments/assets/d7a17536-302d-43b5-9197-519c4784ab26)
# Project Overview:
This SQL project focuses on designing and managing a library database to optimize operations, improve data accessibility, and generate actionable insights. It involves creating, manipulating, and analyzing various tables like books, members, employees, branch performance, overdue books, and more. The project demonstrates the use of SQL for database design, CRUD operations, stored procedures, and advanced queries to derive meaningful insights and automate library processes.

# Objectives:
1.	Build a structured database to manage books, members, employees, and branches efficiently.
2.	Perform CRUD operations (create,read,update,delete) to maintain up-to-date records.
3.	Analyze overdue books, active members, and branch performance to identify areas for improvement.
4.	Automate library operations like book issuance, returns, and overdue management using stored procedures.
5.	Generate summary tables and reports for better decision-making and resource optimization.

# Queries performed:
# 1.Insert sample records into the books table

	insert into books values('978-1-60129-456-2','To Kill a Mockingbird', 'Classic', 6.00, 'yes', 'Harper Lee', 'J.B. Lippincott & Co.')

# 2.Read and display data from different tables

	select * from books
	select * from members

# 3.1.Update an Existing Member's Address

	update members set member_address='125 Oak st'where member_id='C103'

# 3.2.Adding new column in return_status

	alter table return_status add book_quality varchar(15) 
	update return_status set book_quality= 'Good' where book_quality is null
	update return_status set book_quality='Damaged' where issued_id in ('IS112', 'IS117', 'IS118')
	select * from return_status

# 4.Delete a Record from the Issued Status Table 

	delete from issued_status where issued_id='IS101'


# --Data Analysis and Findings
# 1: Retrieve All Books Issued by a Specific Employee 

	select * from issued_status where issued_emp_id='E101'

# 2.List Members Who Have Issued More Than One Book 

 	select issued_emp_id,COUNT(*) as 'Number of books issued' from issued_status
		group by issued_emp_id having count(*)>1

# 3.Create Summary Tables:Generate a new tables based on query results - each book and total book_issued_cnt**

	select b.isbn,b.book_title, count(i.issued_id) as Issued_count into Summary_table from 
		books b inner join issued_status i on b.isbn=i.issued_book_isbn
		group by b.isbn,b.book_title 

	select * from Summary_table


# 4.Retrieve All Books in a Specific Category:

	select * from books where category='Classic'

# 5.Find Total Rental Income by Category.

	select category,SUM(rental_price) * COUNT(*) as Total_rental_income from books group by category

# 6.List Members Who Registered in the Last 180 Days:

	select * from members where reg_date>=DATEADD(day,-180,getdate())

# 7.List Employees with Their Branch Manager's Name and their branch details.

	select e1.emp_name,e1.emp_id,e1.position,e1.salary,b.*,e2.emp_name as manager from 
		employees as e1 inner join branch as b on e1.branch_id = b.branch_id 
		inner join employees as e2 on e2.emp_id = b.manager_id

# 8. Create a Table of Expensive Books with Rental Price Above a Certain Threshold:
	select * into Expensive_books from books where rental_price >7.00
	select * from Expensive_books

# 9. Retrieve the List of Books Not Yet Returned

	select i.issued_book_name as Books_Not_Yet_Returned from issued_status i 
		left join return_status r on i.issued_id=r.issued_id 
		where r.return_id is null

--10.Identify members who have overdue books (assume a 30-day return period). Display the member's_id, member's name, book title, issue date, and days overdue.
select i.issued_member_id,
	   m.member_name,
	   i.issued_book_name,
	   i.issued_date, 
	   DATEDIFF(DAY, i.issued_date, GETDATE()) as Days_Overdue
 	   from members m inner join issued_status i on m.member_id=i.issued_member_id left join return_status r on i.issued_id=r.issued_id
   	   where r.return_id is null and DATEDIFF(DAY, i.issued_date, GETDATE()) > 30

--11.Update the status of books for return in the books table to "Yes" when they are returned (based on entries in the return_status table).
create procedure add_return_records
	@p_return_id varchar(50), 
	@p_issued_id VARCHAR(50), 
	@p_book_quality VARCHAR(15)
as
begin
declare @v_isbn VARCHAR(50),@v_book_name VARCHAR(100)
	insert into return_status(return_id, issued_id, return_date, book_quality)
	values(@p_return_id, @p_issued_id, GETDATE(), @p_book_quality)

	select @v_isbn=issued_book_isbn,@v_book_name=issued_book_name from issued_status
    where issued_id = @p_issued_id;

    update books
    set status = 'yes'
    where isbn= @v_isbn
	print 'Thank you for returning the book: '+@v_book_name
end
go

exec add_return_records 'RS138', 'IS135', 'Good'
exec add_return_records 'RS148', 'IS140', 'Good'

--12.Branch Performance Report
--Create a query that generates a performance report for each branch, showing the number of books issued, the number of books returned, and the total 
--revenue generated from book rentals.
select b.branch_id,COUNT(i.issued_id) as Number_of_books_issued, COUNT(r.return_id) as Number_of_book_returned,SUM(bk.rental_price) as Total_Revenue
	into Branch_performance_report from branch b inner join employees e on b.branch_id=e.branch_id 
	inner join issued_status i on e.emp_id=i.issued_emp_id 
	left join return_status r on i.issued_id=r.issued_id 
	inner join books bk on i.issued_book_isbn=bk.isbn
	group by b.branch_id 

select * from Branch_performance_report

--13.CTAS: Create a Table of Active Members
-- Use CREATE TABLE AS(CTAS) statement to create a new table active_members containing members who have issued at least one book in the last 7 months.
select distinct(m.member_id),m.member_name into Active_Members from 
	members m inner join issued_status i on m.member_id=i.issued_member_id 
	where DATEDIFF(MONTH, i.issued_date, GETDATE()) <=7

select * from Active_Members

--14.Find Employees with the Most Book Issues Processed
-- Write a query to find the top 6 employees who have processed the most book issues.Display the employee name, number of books processed, and their branch.
select top(6)e.emp_name,e.branch_id,COUNT(issued_id) as Number_of_books_issued 
	from issued_status i inner join employees e on i.issued_emp_id=e.emp_id
	group by e.emp_name,e.branch_id order by COUNT(issued_id) desc

--15.Write a stored procedure that updates the status of a book in the library based on its issuance. 
--The stored procedure should take the book_id as an input parameter. 
--The procedure should first check if the book is available (status = 'yes'). 
--If the book is available, it should be issued, and the status in the books table should be updated to 'no'. 
--If the book is not available (status = 'no'), the procedure should return an error message indicating that the book is currently not available.
create procedure book_issued
	@p_issued_id varchar(50),
	@p_issued_member_id varchar(50),
	@p_issued_book_name varchar(100),
	@p_issued_book_isbn varchar(50),
	@p_issued_emp_id varchar(50)
as begin
declare @v_status varchar(50)
	select @v_status=status from books
	where isbn=@p_issued_book_isbn

	if @v_status='yes'
		begin
			insert into issued_status(issued_id,issued_member_id,issued_book_name,issued_date,issued_book_isbn,issued_emp_id)
			values(@p_issued_id,@p_issued_member_id,@p_issued_book_name,GETDATE(),@p_issued_book_isbn,@p_issued_emp_id)
			update books
			set status='no'
			where isbn=@p_issued_book_isbn
			print 'Book records added successfully for book isbn :'+@p_issued_book_isbn
		end
	else
		begin
			print'Sorry to inform you the book you have requested is unavailable book_isbn:'+@p_issued_book_isbn
		end
end
go
exec book_issued'IS150','C107','Where the Wild Things Are','978-0-06-025492-6','E106'
exec book_issued'IS152','C106','The Diary of a Young Girl','978-0-375-41398-8','E105'

--16. Write a CTAS query to create a new table that lists each member and the books they have issued but not returned within 30 days. The table should include: 
--The number of overdue books.The total fines, with each day's fine calculated at $0.50. The number of books issued by each member. The resulting table should show:
--Member ID ,Number of overdue books ,Total fines

select i.issued_member_id,i.issued_book_name,i.issued_date,DATEDIFF(DAY, i.issued_date, GETDATE()) as Days_Overdue into Overdue_books_deatils from issued_status i left join return_status r on i.issued_id=r.issued_id
	where r.return_id is null and DATEDIFF(DAY, i.issued_date, GETDATE()) > 30

select issued_member_id,COUNT(issued_book_name) as Number, SUM(DATEDIFF(DAY,issued_date, GETDATE())*0.50) as Total_fine 
	from Overdue_books_deatils group by issued_member_id

# Key Findings:
1.	 Analyzing issuance data can identify trends in member activity levels and popular book categories. (Queries 1, 2, 6, 13) 
2.	Reports on book issuance and returns enable evaluation of branch performance and resource allocation strategies. (Query 12)
3.	Identifying overdue books and calculating associated fines helps streamline collection processes and potentially generate additional revenue. (Queries 9, 10, 16) 
4.	Analyzing book issuance data processed by employees can provide insights into employee workload and identify top performers. (Query 14)

# Conclusion:
This project showcases the effective use of SQL for managing and analyzing library operations. The structured database, advanced queries, and automated procedures streamline processes like book issuance, overdue tracking, and performance reporting. The insights gained from the analysis enable better resource allocation, operational efficiency, and enhance library management. 
