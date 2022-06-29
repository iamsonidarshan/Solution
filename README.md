# Solution

 - **In this task, you need to review a single Java class that is used for a publicly accessible web application.**
 - **You should add the comments on the problematic code lines, explain with languages that can be understood by software engineers, propose the fix, and add a note for a potential long-term solution if necessary.**
 - **Your answers will be verified and graded manually.**

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.ResultSetExtractor;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.ModelAndView;
import javax.servlet.http.HttpSession;

import java.util.*;

@RequestMapping("/booking")
	@Controller
	class BookingController {

	private final JdbcTemplate jdbcTemplate;

	public BookingController(JdbcTemplate jdbcTemplate) {
	this.jdbcTemplate = jdbcTemplate;
	}

	@RequestMapping("/")
	public ModelAndView home(HttpSession session) {
	long profileId = session.getAttribute("PROFILE_ID");
	List<Booking> bookings = loadBookings(profileId);

	Map<String, Object> model = new HashMap<>();
	model.put("bookings", bookings);

	return new ModelAndView("views/booking/home", model);
	}

	@RequestMapping("/detail")
	public ModelAndView detail(@RequestParam(value = "bookingId") String bookingId) {
	Map<String, Object> model = new HashMap<>();
	if (isNumeric(bookingId)) {
	String sql = "SELECT * FROM bookings WHERE bookingId=" + bookingId;
	
	/*
	
		In the above line of code a SQL query is being generated but because the isNumeric() method is buggy it is a possibility that bookingId can contain special characters which can break the SQL query and this can lead to SQL injection attacks. 

		To solve it for this time and to prevent any future occurance of the same. It is highly recommended to use parameterized query whenever there is a requirement to make SQL queries by appending the user-input. 
	
	*/
	
	Booking booking = null;
	jdbcTemplate.query(sql, (ResultSetExtractor) rs -> {
	if (rs.next())
	booking = new Booking(rs.getLong(1), rs.getString(2), rs.getString(3));
	return null;
	});
	model.put("bookingDetail", booking);
	}
	return new ModelAndView("views/booking/detail", model);
	}

	@RequestMapping("/updateContact")
	public ModelAndView update(@RequestParam(value = "bookingId") String bookingId,
	@RequestParam(value = "contactNumber") String contactNumber)) {
	Map<String, Object> model = new HashMap<>();
	if (isNumeric(bookingId)) {
	String sqlPrepared = "UPDATE bookings SET contactNumber = '%s' WHERE bookingId=" + bookingId; 
	
	/*
		As mentioned above, it is recommended that parameterized queries should be use to pass user inputs to SQL queries. 
	
	*/
	
	String sql = String.format(sqlPrepared, contactNumber);
	jdbcTemplate.update(sql);
	model.put("newContact", contactNumber);
	}
	return new ModelAndView("views/booking/contactUpdated", model);
	}

	private List<Booking> loadBookings(long profileId) {
	return jdbcTemplate.query("SELECT * FROM bookings WHERE owner=" + profileId, rs -> {
	List<Booking> bookings = new LinkedList<>();
	while (rs.next()) {
	bookings.add(new Booking(rs.getLong(1), rs.getString(2), rs.getString(3)));
	}
	return bookings;
	});
	}

	private boolean isNumeric(String strNum) { 
	
	/*
	
	This method is not checking if the given string is a number or not. Insted, the logic is detecting if the string contains a number or not. 
	
	For example, 
	
	This method will return true if the strNum is '123' <- only numbers
	This method will return true if the strNum is 'abcd123' <- numbers and alphabets
	This method will return false if the strNum is 'abcd' <- only alphabets
	
	This method should be fixed and a strict check should be implemented to make sure it retunrs true only if theinput contains numbers and no alphabets.
	
	*/
	if (strNum == null) {
	return false;
	}
	Pattern pattern = Pattern.compile("-?\\d+(\\.\\d+)?");
	return pattern.matcher(strNum).find();
	}
	}
```
