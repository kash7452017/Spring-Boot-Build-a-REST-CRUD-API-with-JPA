## Spring Boot - Build a REST CRUD API with JPA
>參考網址：https://blog.csdn.net/xhaimail/article/details/102582456
>
>JPA和Hibernate之間的最大的區別是：
>
>* JPA是一個規範，不是框架
>* Hibernate是JPA的實現
>
>也可以簡單的理解為JPA是標準接口，Hibernate是實現
>>Hibernate主要是通過三個組件來實現的，即hibernate-annotation、hibernate-entitymanager和hibernate-core。
>>
>>* hibernate-annotation：是Hibernate支持annotation方式配置的基礎，它包括了標準的JPA annotation以及Hibernate自身特殊功能的annotation。
>>* hibernate-core：是Hibernate的核心實現，提供了Hibernate所有的核心功能。
>>* hibernate-entitymanager：實現了標準的JPA，可以把它看成hibernate-core和JPA之間的適配器，它並不直接提供ORM的功能，而是對hibernate-core進行封裝，使得Hibernate符合JPA的規範

**Hibernate / JPA**
```
// Hibernate / JPA
#session.save(...)			entityManager.persist(...) // 創建
#session.get/load(...)		entityManager.find(...)		// 單一查詢
#session.createQuery(...)		entityManager.createQuery(...)	// 全部查詢 
#session.saveOrUpdate(...)	entityManager.merge(...)		// 存取或更新
#session.delete(...)			entityManager.remove(...)		// 刪除
```


### Version 2：Use EntityManager and standard JPA API

**EmployeeDAO.java程式碼，創建DAO方法接口，包含基本CRUD**
```
public interface EmployeeDAO {

	public List<Employee> findAll ();
	
	public Employee findById(int theId);
	
	public void save(Employee theEmployee);
	
	public void deleteById(int theId);
}
```
**EmployeeDAOJpaImpl.java程式碼，透過標準JPA API方式來完成DAO接口方法實現，並透過構造函數方式注入EntityManager**
```
@Repository
public class EmployeeDAOJpaImpl implements EmployeeDAO {

	private EntityManager entityManager;
	
	@Autowired
	public EmployeeDAOJpaImpl(EntityManager theEntityManager) {
		entityManager = theEntityManager;
	}
	
	@Override
	public List<Employee> findAll() {
		
		// create a query
		Query theQuery = 
				entityManager.createQuery("from Employee");
		
		// execute query and get result list
		List<Employee> employees = theQuery.getResultList();
		
		// return the results		
		return employees;
	}

	@Override
	public Employee findById(int theId) {
		
		// get Employee
		Employee theEmployee = 
				entityManager.find(Employee.class, theId);
		
		// return employee		
		return theEmployee;
	}

	@Override
	public void save(Employee theEmployee) {
		
		// save or update the employee
		Employee dbEmployee = entityManager.merge(theEmployee);
		
		// update with id from db ... so we can get generated id for save/insert
		theEmployee.setId(dbEmployee.getId());
	}

	@Override
	public void deleteById(int theId) {
		
		// delete object with primary key
		Query theQuery = entityManager.createQuery(
							"delete from Employee where id=:employeeId"); 
		
		theQuery.setParameter("employeeId", theId);
		
		theQuery.executeUpdate();
	}
}
```
**EmployeeService.java程式碼，同樣創建服務層方法接口，接口方法與DAO相同，僅是透過Service委託調用方法**
```
public interface EmployeeService {

	public List<Employee> findAll();
	
	public Employee findById(int theId);
	
	public void save(Employee theEmployee);
	
	public void daleteById(int theId);
}
```
**EmployeeServiceImpl.java程式碼，完成服務層接口方法實現，實際上僅是委託調用DAO方法實現，因專案中包含兩種DAO方法實現，方便對照比較，因此需透過@Qualifier給定方法實現類別，避免發生錯誤**
```
@Service
public class EmployeeServiceImpl implements EmployeeService {
	
	private EmployeeDAO employeeDAO;
	
	@Autowired
	public EmployeeServiceImpl(@Qualifier("employeeDAOJpaImpl") EmployeeDAO theEmployeeDAO) {
		employeeDAO = theEmployeeDAO;
	}

	@Override
	@Transactional
	public List<Employee> findAll() {		
		return employeeDAO.findAll();
	}

	@Override
	@Transactional
	public Employee findById(int theId) {
		return employeeDAO.findById(theId);
	}

	@Override
	@Transactional
	public void save(Employee theEmployee) {
		employeeDAO.save(theEmployee);
	}

	@Override
	@Transactional
	public void daleteById(int theId) {
		employeeDAO.deleteById(theId);
	}
}
```
**EmployeeRestController.java程式碼，控制器內容不變，僅是更改底層DAO架構，並不影響控制器動作執行方式**
```
@RestController
@RequestMapping("/api")
public class EmployeeRestController {
	
	private EmployeeService employeeService;
	
	@Autowired
	public EmployeeRestController(EmployeeService theEmployeeService) {
		employeeService = theEmployeeService;
	}
	
	// expose "/employees" and return list of employees
	@GetMapping("/employees")
	public List<Employee> findAll(){
		return employeeService.findAll();
	}
	
	// add mapping for GET /employee/{employeeId}
	@GetMapping("/employees/{employeeId}")
	public Employee getEmployee(@PathVariable int employeeId) {
		Employee theEmployee = employeeService.findById(employeeId);
		
		if (theEmployee == null)
			throw new RuntimeException("Employee id not found - " + employeeId);
		
		return theEmployee;
	}
	
	// add mapping for POST /employees - add new employee
	@PostMapping("/employees")
	public Employee addEmployee(@RequestBody Employee theEmployee) {
		
		// also just in case they pass an id in JSON ... set id to 0
		// this is to force a save of new item ... instead of update
		theEmployee.setId(0);
		
		employeeService.save(theEmployee);
		
		return theEmployee;
	}
	
	// add mapping for PUT /employees - update existing employee
	@PutMapping("/employees")
	public Employee updateEmployee(@RequestBody Employee theEmployee) {
		
		employeeService.save(theEmployee);
		
		return theEmployee;
	}
	
	// add mapping for DELETE /employee/{employeeId} - delete employee
	@DeleteMapping("/employees/{employeeId}")
	public String deleteEmployee(@PathVariable int employeeId) {
		
		Employee tempEployee = employeeService.findById(employeeId);
		
		// throw exception if null
		if (tempEployee == null)
			throw new RuntimeException("Employee id not found - " + employeeId);
		
		employeeService.daleteById(employeeId);
		
		return "Deleted emploee id - " + employeeId;
	}
}
```
