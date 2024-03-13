## validation自定义注解
```code
/**
 * @author zrh
 * @description 校验是否不等于某值
 * @date 2021/8/27 10:20
 */
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = NotEqualsValidator.class)
@Documented
public @interface NotEquals {
    String message() default "请传入符合要求的值！";

    String [] value() default "";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}

```
## 注解实现
```code
public class NotEqualsValidator implements ConstraintValidator<NotEquals, String> {
    String[] value = null;
    String message = null;

    @Override
    public void initialize(NotEquals constraintAnnotation) {
        value = constraintAnnotation.value();
        message = constraintAnnotation.message();
    }
    @Override
    public boolean isValid(String str, ConstraintValidatorContext constraintValidatorContext) {
        for (String s : value) {
            if (s.equals(str)) {
                return false;
            }
        }
        return true;
    }
}
```
## 使用
```code
@Data
public class AccountPeriodsConvertModel {
    /**
     * 天数
     */
    @NotBlank(message = "赎期不能为空！")
    private String days;
    /**
     * 百分比
     */
    @NotBlank(message = "比例不能为空！")
    @NotEquals(message = "比例不能为0！",value = {"0","0%"})
    private String percentage;

}

```