####Use InputFW permission check   
___
#####Installation   
1. add dependency on the latest version of InputFW library   
```xml
<dependency>
  <groupId>com.worksap.company</groupId>
  <artifactId>hue-application-input-framework</artifactId>
</dependency>
```
2. import InputFW's config;   
```
 com.worksap.company.framework.inputfw.config.InputFrameworkConfig
```
or only the permission related config:   
```
com.worksap.company.framework.inputfw.permission.PermissionConfig
```
#####Usage   
_you need only care:_   
 * `PermissionService`
 * `Permission`
 * `@RequiresPermission`
 * `@CheckPermission` 

all the 4 files are in package `com.worksap.company.framework.inputfw.permission`   

######`Permission`   
**This interface declares all the permission constants**   

we recommend you extends `com.worksap.company.framework.inputfw.permission.Permission` and make your own PermissionConstants.  

* Normally all the permission constants should consist of two parts(_PermissionSetName_ and _PermissionName_) and be in the form of   
 **_`PermissionSetName.PermissionName`_**, e.g. the `Permission.InputFramework.OWNER`(value is `"InputFramework.Owner"`) in the code below.  
 This kind of permissions are checked by the `PermssionService`-Impl corresponding to the _PermissionSetName_. e.g. permission `"InputFramework.Owner"`  
 is checked by a `PermissionService`-Impl whose `getPermissionSet()` method returns `"InputFramework"`.
 
* But there is also another kind of permission constants named __, which has no `PermissionSetName`, e.g. the `Permission.Application.DOWNLOAD` in the  
 code below.  
 This kind of permissions are checked by the `PermssionService`-Impl corresponding to the `ServiceDefId` to which the checked _Application_ belongs.  
 e.g. when downloading report of _ApplicationXXX_, and _ApplicationXXX_'s `ServiceDefId` is _ServiceDefIdYYY_. Permission `"Download"` will be  
 checked by a `PermissionService`-Impl whose `getServiceDefIds()`'s result contains _ServiceDefIdYYY_.

```java
public interface Permission {
    interface InputFramework {
        String PERMISSION_SET = "InputFramework";
        String APPROVER = PERMISSION_SET + ".Approver"; //InputFramework.Approver
        String OWNER = PERMISSION_SET + ".Owner"; //InputFramework.Owner
    }
    interface Application{
        String DOWNLOAD = "Download"; // has no `PermissionSetName`
    }
}
```
######`PermissionService`   
**Implement this service to check permissions.**      

If you define your own permission set and permissions, you must implement this interface to check the  
permissions you defined. Otherwise, those permissions may not be checked.

If you define permissions which have no _PermissionSetName_, you have to check it by yourself in this `PermssionService`-impl, or you may need to ask  
other products to check at the same time in some cases.

```java
public interface PermissionService extends Service {
    /**
     * get the `PermissionSetName` of the permission set which will be checked by this `PermissionService`
     *
     * @return the `PermissionSetName`
     */
    String getPermissionSet();

    /**
     * @return Collection of `ServiceDefId`
     */
    Collection<ServiceDefId> getServiceDefIds();

    /**
     * check whether user with `userId` has `permissions` on `target` to call method `methodName`
     *
     * @param permissions the required permissions to call method `methodName`, the `PermissionSetName` are not contained.
     * @param args args passed when call method `methodName`
     * @return <code>true</code> if has the required permissions.
     */
    boolean hasPermission(Object target, UserId userId, Collection<String> permissions, String methodName, Object... args);
}

@Slf4j
public class InputFwPermissionServiceImpl implements PermissionService {
    @Override
    public String getPermissionSet() {
        return Permission.InputFramework.PERMISSION_SET;
    }
    
    @Override
    public Collection<ServiceDefId> getServiceDefIds() {
        return Collections.singleton();
    }

    @Override
    public boolean hasPermission(Object target, UserId userId, Collection<String> permissions, String methodName, Object... args) {
        log.info("user[{}] is now requiring permissions[{}.{}] to access application[{}] from method[{}]",
                userId, this.getPermissionSet(), permissions, target, methodName
        );
        return true;
    }
}

@Slf4j
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class AcExpensePermissionServiceImpl implements PermissionService {
    private final ReportAccessSecurityResolver reportSecurityResolver;
    private final ApplicationService<ApplicationContents> applicationService;
    private final UserContext userContext;

    public String getPermissionSet() {
        return Permission.PERMISSION_SET;
    }

    @Override
    public Collection<ServiceDefId> getServiceDefIds() {
        return Arrays.stream(ExpenseServiceDefId.values()).map(ExpenseServiceDefId::toDefId).collect(Collectors.toSet());
    }

    public boolean hasPermission(Object target, UserId userId, Collection<String> permissions, String methodName, Object... args) {
        log.info("user[{}] is now requiring permissions[{}.{}] to access application[{}] from method[{}]",
                userId, this.getPermissionSet(), permissions, target, methodName
        );
        if (permissions.contains(Permission.Application.DOWNLOAD)) {
            UserContextVo contextVo = UserContextVo.valueOf(userContext);
            Application application = applicationService.get(contextVo,
                    ApplicationId.valueOf(String.valueOf(target)), ApplicationPurpose.INSTA_REPORT);
            ReportAccessSecurity security = reportSecurityResolver.resolve(application.getServiceId());
            security.checkSecurity(application.getServiceId(), userContext.getUser(), application);
        }
        return true;
    }
}
```
######`@RequiresPermission`   
**Add this annotation on any method(include `abstract`, `default` or `concrete`/`implementation` methods) to declare which permissions are needed to  
call this method**   
`@RequiresPermission` has 4 properties:
* `value`: same as `permssions`   
* `permissions`: permissions needed to call this method.   
* `callSuper`: whether to include the permissions declared on `overridden` methods, `true` in default. e.g. in the code below, all  
 of `Permission.Application.DOWNLOAD`, `Permission.InputFramework.APPROVER` and `Permission.InputFramework.OWNER` are to be checked.   
* `target`: the target, `"2"` means the third parameter, `"2.text"` means the `text` property of the third parameter, `"2.aaa.bbb..."`

```java
public interface DownloadController extends Controller{
    @RequiresPermission(value = { 
            Permission.Application.DOWNLOAD }, target = "2")
    JasperResponse downloadPdf(String featureName, String reportXmlName, String applicationId, String serviceId);    
}

@CheckPermission
public class ApplicationReportController implements DownloadController {
    @RequiresPermission(value = {
            Permission.InputFramework.APPROVER,
            Permission.InputFramework.OWNER}, target = "2", callSuper = true)
    public JasperResponse downloadPdf(@RequestParam String featureName, @RequestParam String reportXmlName,
            @RequestParam String applicationId, @RequestParam String serviceId) {
        //...
    }
}
```

for the code above, if the `downloadPdf` method is called with parameter `applicationId="appIdxxx"` and this application is an _Ac-Expense_ application,  
then the permission checking result should be:   
```java
String methodName = "package.ApplicationReportController.downloadPdf";
AcExpensePermissionServiceImpl.hasPermission("appIdxxx", userId, {"Downlaod"}, methodName, args)
    ||InputFwPermissionServiceImpl.hasPermission("appIdxxx", userId, {"Approver", "Owner"}, methodName, args)
```
######`@CheckPermission`   
**Add this annotation on any class to activate permission check, permission will not be checked otherwise**   
So, adding this annotation on `interface`, `enum`, `@interface` will have no effect   

```java
@CheckPermission
public class ApplicationReportController implements DownloadController {
    //...
}
```
####Default Permission Control in InputFW    
| Operation         | Permitted User  | Class                         |
| ----------------- | --------------- |: ---------------------------- |
| `start`           | x               | `StartApplicationPageSupport` |
| `restart`         | Applicant       | `StartApplicationPageSupport` |
| `confirm`         | x               | `InputPageSupport`            |
| `finish`          | x               | `InputPageSupport`            |
| `onChange`        | x               | `InputPageSupport`            |
| `save`            | x               | `InputPageSupport`            |
| `copy`            | Applicant       | `CopyPageSupport`             |
| `reference`       | Applicant       | `ReferencePageSupport`        |
| `useDraftApp`     | Applicant       | `PreparedPageSupport`         |
| `useGeneratedApp` | Applicant       | `PreparedPageSupport`         |
| `backHistory`     | x               | `HistoryPageSupport`          |
| `nextHistory`     | x               | `HistoryPageSupport`          |
| `copyHistory`     | x               | `HistoryPageSupport`          |
| `checkHasHistory` | x               | `HistoryPageSupport`          |
