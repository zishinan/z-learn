# mybatis入门与精通-TypeHandler详解
mybatis为我们定义了很多自定义的Handler，以enum这种特殊类型来讲解下如何使用，其他类型的类似处理即可。
#### EnumTypeHandler
mybatis默认处理enum的Handler，将enum的属性名映射到数据库中，处理为字符串。

如果没有配置typeHandler,遇到enum类型的属性就会用EnumTypeHandler处理，要用其他的需要配置：
```xml
<configuration>
    <typeHandlers>
        <typeHandler handler="com.demo.test.service.impl.dao.handler.IdEnumHandler"
                     javaType="com.demo.test.service.type.Module" />
    </typeHandlers>
</configuration>
```
* 值得注意的是，这里如果配置了jdbcType，preperStatement时就不会生效了，还是会采用EnumTypeHandler处理。（原因暂时没深究）
#### EnumOrdinalTypeHandler
mybatis提供的另一个处理enum的Handler，将enum的定义顺序下标映射到数据库中，处理为整型。
#### 自定义EnumHandler
我们也可以继承BaseTypeHandler实现自己的Handler，
#### 自定义通用EnumHandler
首先定义一个接口，让所有要使用这个Handler的enum都实现这个接口。
```java
public interface IdEnum {
    Integer getId();
}
```
然后可以自定义Handler，主要思路来源于EnumOrdinalTypeHandler,只是他是根据enum的顺序获得，而我们要定义为根据某个特殊字段获得，所以将他的enum数组改成Map方便处理。看下他的源码：
```java
public class EnumOrdinalTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

  private Class<E> type;
  private final E[] enums;

  public EnumOrdinalTypeHandler(Class<E> type) {
    if (type == null) throw new IllegalArgumentException("Type argument cannot be null");
    this.type = type;
    /*这行代码是注入数组，我们改为Map，再根据需求确定key*/
    this.enums = type.getEnumConstants();
    if (this.enums == null) throw new IllegalArgumentException(type.getSimpleName() + " does not represent an enum type.");
  }
}
```
自定义通用Handler代码如下：
```java
public class IdEnumHandler<E extends Enum<E>> extends BaseTypeHandler<E> {
    private Class<E> type;
    private Map<Integer, E> map = new HashMap<Integer,E>();
    public IdEnumHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
        E[] enums = type.getEnumConstants();
        if (enums == null) {
            throw new IllegalArgumentException(type.getSimpleName() + " does not represent an enum type.");
        }
        for (E e : enums) {
            IdEnum idEnum = (IdEnum) e;
            map.put(idEnum.getId(), e);
        }
    }
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
        IdEnum idEnum = (IdEnum) parameter;
        ps.setInt(i, idEnum.getId());
    }
    @Override
    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
        int i = rs.getInt(columnName);
        if (rs.wasNull()) {
            return null;
        } else {
            return getIdEnum(i);
        }
    }
    @Override
    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        int i = rs.getInt(columnIndex);
        if (rs.wasNull()) {
            return null;
        } else {
            return getIdEnum(i);
        }
    }
    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        int i = cs.getInt(columnIndex);
        if (cs.wasNull()) {
            return null;
        } else {
            return getIdEnum(i);
        }
    }
    private E getIdEnum(int id) {
        try {
            return map.get(id);
        } catch (Exception ex) {
            throw new IllegalArgumentException(
                    "Cannot convert " + id + " to " + type.getSimpleName() + " by value.", ex);
        }
    }
}
```

