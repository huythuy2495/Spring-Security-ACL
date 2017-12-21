# Spring-Security-ACL
 Introduction

Access Control List (ACL) is a list of permissions attached to an object. An ACL specifies which identities are granted which operations on a given object.

Spring Security Access Control List is a Spring component which supports Domain Object Security. Simply put, Spring ACL helps in defining permissions for specific user/role on a single domain object – instead of across the board, at the typical per-operation level.

For example, a user with the role Admin can see (READ) and edit (WRITE) all messages on a Central Notice Box, but the normal user only can see messages, relate to them and cannot edit. Meanwhile, others user with the role Editor can see and edit some specific messages.

Hence, different user/role has different permission for each specific object. In this case, Spring ACL is capable of achieving the task. We’ll explore how to set up basic permission checking with Spring ACL in this article.

2. Configuration

2.1. ACL Database

To use Spring Security ACL, we need to create four mandatory tables in our database.

The first table is ACL_CLASS, which store class name of the domain object, columns include:

ID
CLASS: the class name of secured domain objects, for example: org.baeldung.acl.persistence.entity.NoticeMessage
Secondly, we need the ACL_SID table which allows us to universally identify any principle or authority in the system. The table needs:

ID
SID: which is the username or role name. SID stands for Security Identity
PRINCIPAL: 0 or 1, to indicate that the corresponding SID is a principal (user, such as mary, mike, jack…) or an authority (role, such as ROLE_ADMIN, ROLE_USER, ROLE_EDITOR…)
Next table is ACL_OBJECT_IDENTITY, which stores information for each unique domain object:

ID
OBJECT_ID_CLASS: define the domain object class, links to ACL_CLASS table
OBJECT_ID_IDENTITY: domain objects can be stored in many tables depending on the class. Hence, this field store the target object primary key
PARENT_OBJECT: specify parent of this Object Identity within this table
OWNER_SID: ID of the object owner, links to ACL_SID table
ENTRIES_INHERITTING: whether ACL Entries of this object inherits from the parent object (ACL Entries are defined in ACL_ENTRY table)
Finally, the ACL_ENTRY store individual permission assigns to each SID on an Object Identity:

ID
ACL_OBJECT_IDENTITY: specify the object identity, links to ACL_OBJECT_IDENTITY table
ACL_ORDER: the order of current entry in the ACL entries list of corresponding Object Identity
SID: the target SID which the permission is granted to or denied from, links to ACL_SID table
MASK: the integer bit mask that represents the actual permission being granted or denied
GRANTING: value 1 means granting, value 0 means denying
AUDIT_SUCCESS and AUDIT_FAILURE: for auditing purpose
2.2. Dependency

To be able to use Spring ACL in our project, let’s first define our dependencies:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-acl</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
</dependency>
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache-core</artifactId>
    <version>2.6.11</version>
</dependency>
Spring ACL requires a cache to store Object Identity and ACL entries, so we’ll make use of Ehcache here. And, to support Ehcache in Spring, we also need the spring-context-support.

When not working with Spring Boot, we need to add versions explicitly. Those can be checked on Maven Central: spring-security-acl, spring-security-config, spring-context-support, ehcache-core.

2.3. ACL-Related Configuration

We need to secure all methods which return secured domain objects, or make changes to the object, by enabling Global Method Security:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class AclMethodSecurityConfiguration 
  extends GlobalMethodSecurityConfiguration {
 
    @Autowired
    MethodSecurityExpressionHandler 
      defaultMethodSecurityExpressionHandler;
 
    @Override
    protected MethodSecurityExpressionHandler createExpressionHandler() {
        return defaultMethodSecurityExpressionHandler;
    }
}
Let’s also enable Expression-Based Access Control by setting prePostEnabled to true to use Spring Expression Language (SpEL). Moreover, we need an expression handler with ACL support:

1
2
3
4
5
6
7
8
9
10
@Bean
public MethodSecurityExpressionHandler 
  defaultMethodSecurityExpressionHandler() {
    DefaultMethodSecurityExpressionHandler expressionHandler
      = new DefaultMethodSecurityExpressionHandler();
    AclPermissionEvaluator permissionEvaluator 
      = new AclPermissionEvaluator(aclService());
    expressionHandler.setPermissionEvaluator(permissionEvaluator);
    return expressionHandler;
}
Hence, we assign AclPermissionEvaluator to the DefaultMethodSecurityExpressionHandler. The evaluator needs a MutableAclService to load permission settings and domain object’s definitions from the database.

For simplicity, we use the provided JdbcMutableAclService:

1
2
3
4
5
@Bean 
public JdbcMutableAclService aclService() { 
    return new JdbcMutableAclService(
      dataSource, lookupStrategy(), aclCache()); 
}
As its name, the JdbcMutableAclService uses JDBCTemplate to simplify database access. It needs a DataSource (for JDBCTemplate), LookupStrategy (provides an optimized lookup when querying the database), and an AclCache (caching ACL Entries and Object Identity).

Again, for simplicity, we use provided BasicLookupStrategy and EhCacheBasedAclCache.

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
@Autowired
DataSource dataSource;
 
@Bean
public AclAuthorizationStrategy aclAuthorizationStrategy() {
    return new AclAuthorizationStrategyImpl(
      new SimpleGrantedAuthority("ROLE_ADMIN"));
}
 
@Bean
public PermissionGrantingStrategy permissionGrantingStrategy() {
    return new DefaultPermissionGrantingStrategy(
      new ConsoleAuditLogger());
}
 
@Bean
public EhCacheBasedAclCache aclCache() {
    return new EhCacheBasedAclCache(
      aclEhCacheFactoryBean().getObject(), 
      permissionGrantingStrategy(), 
      aclAuthorizationStrategy()
    );
}
 
@Bean
public EhCacheFactoryBean aclEhCacheFactoryBean() {
    EhCacheFactoryBean ehCacheFactoryBean = new EhCacheFactoryBean();
    ehCacheFactoryBean.setCacheManager(aclCacheManager().getObject());
    ehCacheFactoryBean.setCacheName("aclCache");
    return ehCacheFactoryBean;
}
 
@Bean
public EhCacheManagerFactoryBean aclCacheManager() {
    return new EhCacheManagerFactoryBean();
}
 
@Bean
public LookupStrategy lookupStrategy() { 
    return new BasicLookupStrategy(
      dataSource, 
      aclCache(), 
      aclAuthorizationStrategy(), 
      new ConsoleAuditLogger()
    ); 
}
Here, the AclAuthorizationStrategy is in charge of concluding whether a current user possesses all required permissions on certain objects or not.

It needs the support of PermissionGrantingStrategy, which defines the logic for determining whether a permission is granted to a particular SID.

3. Method Security With Spring ACL

So far, we’ve done all necessary configuration. Now we can put required checking rule on our secured methods.

By default, Spring ACL refers to BasePermission class for all available permissions. Basically, we have a READ, WRITE, CREATE, DELETE and ADMINISTRATION permission.

Let’s try to define some security rules:

1
2
3
4
5
6
7
8
@PostFilter("hasPermission(filterObject, 'READ')")
List<NoticeMessage> findAll();
     
@PostAuthorize("hasPermission(returnObject, 'READ')")
NoticeMessage findById(Integer id);
     
@PreAuthorize("hasPermission(#noticeMessage, 'WRITE')")
NoticeMessage save(@Param("noticeMessage")NoticeMessage noticeMessage);
After the execution of findAll() method, @PostFilter will be triggered. The required rule hasPermission(filterObject, ‘READ’), means returning only those NoticeMessage which current user has READ permission on.

Similarly, @PostAuthorize is triggered after the execution of findById() method, make sure only return the NoticeMessage object if the current user has READ permission on it. If not, the system will throw an AccessDeniedException.

On the other side, the system triggers the @PreAuthorize annotation before invoking the save() method. It will decide where the corresponding method is allowed to execute or not. If not, AccessDeniedException will be thrown.

4. In Action

Now we gonna test all those configurations using JUnit. We’ll use H2 database to keep configuration as simple as possible.

We’ll need to add:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
</dependency>
 
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-test</artifactId>
  <scope>test</scope>
</dependency>
 
<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-test</artifactId>
  <scope>test</scope>
</dependency>
4.1. The Scenario

In this scenario, we’ll have two users (manager, hr) and a one user role (ROLE_EDITOR), so our acl_sid will be:

1
2
3
4
INSERT INTO acl_sid (id, principal, sid) VALUES
  (1, 1, 'manager'),
  (2, 1, 'hr'),
  (3, 0, 'ROLE_EDITOR');
Then, we need to declare NoticeMessage class in acl_class. And three instances of NoticeMessage class will be inserted in system_message. 

Moreover, corresponding records for those 3 instances must be declared in acl_object_identity:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
INSERT INTO acl_class (id, class) VALUES
  (1, 'org.baeldung.acl.persistence.entity.NoticeMessage');
 
INSERT INTO system_message(id,content) VALUES
  (1,'First Level Message'),
  (2,'Second Level Message'),
  (3,'Third Level Message');
 
INSERT INTO acl_object_identity 
  (id, object_id_class, object_id_identity, 
  parent_object, owner_sid, entries_inheriting) 
  VALUES
  (1, 1, 1, NULL, 3, 0),
  (2, 1, 2, NULL, 3, 0),
  (3, 1, 3, NULL, 3, 0);
Initially, we grant READ and WRITE permissions on the first object (id =1) to the user manager. Meanwhile, any user with ROLE_EDITOR will have READ permission on all three objects but only possess WRITE permission on the third object (id=3). Besides, user hr will have only READ permission on the second object.

Here, because we use default Spring ACL BasePermission class for permission checking, the mask value of the READ permission will be 1, and the mask value of WRITE permission will be 2. Our data in acl_entry will be:

1
2
3
4
5
6
7
8
9
10
11
INSERT INTO acl_entry 
  (id, acl_object_identity, ace_order, 
  sid, mask, granting, audit_success, audit_failure) 
  VALUES
  (1, 1, 1, 1, 1, 1, 1, 1),
  (2, 1, 2, 1, 2, 1, 1, 1),
  (3, 1, 3, 3, 1, 1, 1, 1),
  (4, 2, 1, 2, 1, 1, 1, 1),
  (5, 2, 2, 3, 1, 1, 1, 1),
  (6, 3, 1, 3, 1, 1, 1, 1),
  (7, 3, 2, 3, 2, 1, 1, 1);
4.2. Test Case

First of all, we try to call the findAll method.

As our configuration, the method returns only those NoticeMessage on which the user has READ permission.

Hence, we expect the result list contains only the first message:

1
2
3
4
5
6
7
8
9
10
@Test
@WithMockUser(username = "manager")
public void
  givenUserManager_whenFindAllMessage_thenReturnFirstMessage(){
    List<NoticeMessage> details = repo.findAll();
  
    assertNotNull(details);
    assertEquals(1,details.size());
    assertEquals(FIRST_MESSAGE_ID,details.get(0).getId());
}
Then we try to call the same method with any user which has the role – ROLE_EDITOR. Note that, in this case, these users have the READ permission on all three objects.

Hence, we expect the result list will contain all three messages:

1
2
3
4
5
6
7
8
9
@Test
@WithMockUser(roles = {"EDITOR"})
public void 
  givenRoleEditor_whenFindAllMessage_thenReturn3Message(){
    List<NoticeMessage> details = repo.findAll();
     
    assertNotNull(details);
    assertEquals(3,details.size());
}
Next, using the manager user, we’ll try to get the first message by id and update its content – which should all work fine:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
@Test
@WithMockUser(username = "manager")
public void 
  givenUserManager_whenFind1stMessageByIdAndUpdateItsContent_thenOK(){
    NoticeMessage firstMessage = repo.findById(FIRST_MESSAGE_ID);
    assertNotNull(firstMessage);
    assertEquals(FIRST_MESSAGE_ID,firstMessage.getId());
         
    firstMessage.setContent(EDITTED_CONTENT);
    repo.save(firstMessage);
         
    NoticeMessage editedFirstMessage = repo.findById(FIRST_MESSAGE_ID);
  
    assertNotNull(editedFirstMessage);
    assertEquals(FIRST_MESSAGE_ID,editedFirstMessage.getId());
    assertEquals(EDITTED_CONTENT,editedFirstMessage.getContent());
}
But if any user with the ROLE_EDITOR role updates the content of the first message – our system will throw an AccessDeniedException:

1
2
3
4
5
6
7
8
9
10
11
12
@Test(expected = AccessDeniedException.class)
@WithMockUser(roles = {"EDITOR"})
public void 
  givenRoleEditor_whenFind1stMessageByIdAndUpdateContent_thenFail(){
    NoticeMessage firstMessage = repo.findById(FIRST_MESSAGE_ID);
  
    assertNotNull(firstMessage);
    assertEquals(FIRST_MESSAGE_ID,firstMessage.getId());
  
    firstMessage.setContent(EDITTED_CONTENT);
    repo.save(firstMessage);
}
Similarly, the hr user can find the second message by id, but will fail to update it:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
@Test
@WithMockUser(username = "hr")
public void givenUsernameHr_whenFindMessageById2_thenOK(){
    NoticeMessage secondMessage = repo.findById(SECOND_MESSAGE_ID);
    assertNotNull(secondMessage);
    assertEquals(SECOND_MESSAGE_ID,secondMessage.getId());
}
 
@Test(expected = AccessDeniedException.class)
@WithMockUser(username = "hr")
public void givenUsernameHr_whenUpdateMessageWithId2_thenFail(){
    NoticeMessage secondMessage = new NoticeMessage();
    secondMessage.setId(SECOND_MESSAGE_ID);
    secondMessage.setContent(EDITTED_CONTENT);
    repo.save(secondMessage);
}
5. Conclusion

We’ve gone through basic configuration and usage of Spring ACL in this article.

As we know, Spring ACL required specific tables for managing object, principle/authority, and permission setting. All interactions with those tables, especially updating action, must go through AclService. We’ll explore this service for basic CRUD actions in a future article.

By default, we are restricted to predefined permission in BasePermission class
