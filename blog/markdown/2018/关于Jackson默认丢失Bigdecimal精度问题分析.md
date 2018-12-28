##��������
�����ʹ��һ���ڲ���RPC���ʱ���������ʹ��Object���ͣ�ʵ������ΪBigDecimal��ʱ����Ϊ��������ʱ�򣬻���ֶ�ʧ���ȵ����⣻���������л�ǰΪ���1.00�������л�֮��Ϊ1.0������ֵ����û��Ӱ�죬��������Щǿ�������ĵط�����������⣻

##�������
�鿴Դ�뷢��RPC���Ĭ��ʹ�õ����л����ΪJackson���Ǽ򵥣���һ�±����Ƿ�����������⣻
###1.׼�����ݴ���bean

```
public class Bean1 {
 
    private String p1;
    private BigDecimal p2;
     
    ...ʡ��get/set...
}
 
public class Bean2 {
 
    private String p1;
    private Object p2;
     
    ...ʡ��get/set...
}
```
Ϊ�˸��õĿ������⣬�ֱ�׼����2��bean��

###2.׼��������

```
public class JKTest {
 
    public static void main(String[] args) throws IOException {
        ObjectMapper mapper = new ObjectMapper();
 
        Bean1 bean1 = new Bean1("haha1", new BigDecimal("1.00"));
        Bean2 bean2 = new Bean2("haha2", new BigDecimal("2.00"));
 
        String bs1 = mapper.writeValueAsString(bean1);
        String bs2 = mapper.writeValueAsString(bean2);
 
        System.out.println(bs1);
        System.out.println(bs2);
 
        Bean1 b1 = mapper.readValue(bs1, Bean1.class);
        System.out.println(b1.toString());
         
        Bean2 b22 = mapper.readValue(bs2, Bean2.class);
        System.out.println(b22.toString());
    }
}
```
�ֱ��Bean1��Bean2�������л��ͷ����л�������Ȼ��鿴�����

###3.��ʾ���

```
{"p1":"haha1","p2":1.00}
{"p1":"haha2","p2":2.00}
Bean1 [p1=haha1, p2=1.00]
Bean2 [p1=haha2, p2=2.0]
```
###4.�������
������Է����������⣺
1.�����л���ʱ��2��bean��û�����⣻
2.���������⣬Bean2�ڷ����л�ʱ��p2�����˾��ȶ�ʧ�����⣻

###5.Դ�����
ͨ��һ��һ���鿴JacksonԴ�룬���ն�λ��UntypedObjectDeserializer��Vanilla�ڲ����У������з������£�

```
public Object deserialize(JsonParser p, DeserializationContext ctxt) throws IOException
        {
            switch (p.getCurrentTokenId()) {
            case JsonTokenId.ID_START_OBJECT:
                {
                    JsonToken t = p.nextToken();
                    if (t == JsonToken.END_OBJECT) {
                        return new LinkedHashMap<String,Object>(2);
                    }
                }
            case JsonTokenId.ID_FIELD_NAME:
                return mapObject(p, ctxt);
            case JsonTokenId.ID_START_ARRAY:
                {
                    JsonToken t = p.nextToken();
                    if (t == JsonToken.END_ARRAY) { // and empty one too
                        if (ctxt.isEnabled(DeserializationFeature.USE_JAVA_ARRAY_FOR_JSON_ARRAY)) {
                            return NO_OBJECTS;
                        }
                        return new ArrayList<Object>(2);
                    }
                }
                if (ctxt.isEnabled(DeserializationFeature.USE_JAVA_ARRAY_FOR_JSON_ARRAY)) {
                    return mapArrayToArray(p, ctxt);
                }
                return mapArray(p, ctxt);
            case JsonTokenId.ID_EMBEDDED_OBJECT:
                return p.getEmbeddedObject();
            case JsonTokenId.ID_STRING:
                return p.getText();
 
            case JsonTokenId.ID_NUMBER_INT:
                if (ctxt.hasSomeOfFeatures(F_MASK_INT_COERCIONS)) {
                    return _coerceIntegral(p, ctxt);
                }
                return p.getNumberValue(); // should be optimal, whatever it is
 
            case JsonTokenId.ID_NUMBER_FLOAT:
                if (ctxt.isEnabled(DeserializationFeature.USE_BIG_DECIMAL_FOR_FLOATS)) {
                    return p.getDecimalValue();
                }
                return p.getNumberValue();
 
            case JsonTokenId.ID_TRUE:
                return Boolean.TRUE;
            case JsonTokenId.ID_FALSE:
                return Boolean.FALSE;
 
            case JsonTokenId.ID_END_OBJECT:
                // 28-Oct-2015, tatu: [databind#989] We may also be given END_OBJECT (similar to FIELD_NAME),
                //    if caller has advanced to the first token of Object, but for empty Object
                return new LinkedHashMap<String,Object>(2);
 
            case JsonTokenId.ID_NULL: // 08-Nov-2016, tatu: yes, occurs
                return null;
 
            //case JsonTokenId.ID_END_ARRAY: // invalid
            default:
            }
            return ctxt.handleUnexpectedToken(Object.class, p);
        }
```
��Bean2�е�p2��һ��Object���ͣ�����Jackson�и����ķ����л���ΪUntypedObjectDeserializer������Ƚ�������⣻Ȼ����ݾ�����������ͣ����ò��õĶ�ȡ��������Ϊjson�������л���ʽ���������ݣ�����û�д�ž�����������ͣ���������Jackson�϶�2.00Ϊһ��ID_NUMBER_FLOAT���ͣ������case������2��ѡ��Ĭ����ֱ�ӵ���getNumberValue()��������������ᶪʧ���ȣ����ؽ��Ϊ2.0�����߿���ʹ��USE_BIG_DECIMAL_FOR_FLOATS���ԣ�������Ҳ�ܼ򵥣�ʹ�ô����Լ��ɣ�

###6.ʹ��USE_BIG_DECIMAL_FOR_FLOATS����

```
ObjectMapper mapper = new ObjectMapper();
mapper.enable(DeserializationFeature.USE_BIG_DECIMAL_FOR_FLOATS);
```
�ٴβ��ԣ����Է��ֽ�����£�

```
{"p1":"haha1","p2":1.00}
{"p1":"haha2","p2":2.00}
Bean1 [p1=haha1, p2=1.00]
Bean2 [p1=haha2, p2=2.00]
```
###7.��������չ
Jackson�����ṩ�˶����л��ͷ�������չ�Ĺ��ܣ���Ӧ�����Bean�����Լ����巴�����࣬�������Bean2������ʵ��Bean2Deserializer��Ȼ����ObjectMapper����ע��

```
ObjectMapper mapper = new ObjectMapper();
SimpleModule desModule = new SimpleModule("testModule");
desModule.addDeserializer(Bean2.class, new Bean2Deserializer(Bean2.class));
mapper.registerModule(desModule);
```

##��չ
Json����û�д���������ͣ�ֻ�����ݱ�����Ӧ����Json�����л���ʽӦ�ö����ڴ����⣻
###1.FastJson����
׼�����Դ������£�

```
public class FJTest {
 
    public static void main(String[] args) {
        Bean1 bean1 = new Bean1("haha1", new BigDecimal("1.00"));
        Bean2 bean2 = new Bean2("haha2", new BigDecimal("2.00"));
 
        String jsonString1 = JSON.toJSONString(bean1);
        String jsonString2 = JSON.toJSONString(bean2);
 
        System.out.println(jsonString1);
        System.out.println(jsonString2);
 
        Bean1 bean11 = JSON.parseObject(jsonString1, Bean1.class);
        Bean2 bean22 = JSON.parseObject(jsonString2, Bean2.class);
 
        System.out.println(bean11.toString());
        System.out.println(bean22.toString());
 
    }
 
}
```
������£�

```
{"p1":"haha1","p2":1.00}
{"p1":"haha2","p2":2.00}
Bean1 [p1=haha1, p2=1.00]
Bean2 [p1=haha2, p2=2.00]
```
���Է���FastJson�������ڴ����⣬�鿴Դ�룬��λ��DefaultJSONParser��parse���������ִ������£�

```
public Object parse(Object fieldName) {
        final JSONLexer lexer = this.lexer;
        switch (lexer.token()) {
            case SET:
                lexer.nextToken();
                HashSet<Object> set = new HashSet<Object>();
                parseArray(set, fieldName);
                return set;
            case TREE_SET:
                lexer.nextToken();
                TreeSet<Object> treeSet = new TreeSet<Object>();
                parseArray(treeSet, fieldName);
                return treeSet;
            case LBRACKET:
                JSONArray array = new JSONArray();
                parseArray(array, fieldName);
                if (lexer.isEnabled(Feature.UseObjectArray)) {
                    return array.toArray();
                }
                return array;
            case LBRACE:
                JSONObject object = new JSONObject(lexer.isEnabled(Feature.OrderedField));
                return parseObject(object, fieldName);
            case LITERAL_INT:
                Number intValue = lexer.integerValue();
                lexer.nextToken();
                return intValue;
            case LITERAL_FLOAT:
                Object value = lexer.decimalValue(lexer.isEnabled(Feature.UseBigDecimal));
                lexer.nextToken();
                return value;
            case LITERAL_STRING:
                String stringLiteral = lexer.stringVal();
                lexer.nextToken(JSONToken.COMMA);
 
                if (lexer.isEnabled(Feature.AllowISO8601DateFormat)) {
                    JSONScanner iso8601Lexer = new JSONScanner(stringLiteral);
                    try {
                        if (iso8601Lexer.scanISO8601DateIfMatch()) {
                            return iso8601Lexer.getCalendar().getTime();
                        }
                    } finally {
                        iso8601Lexer.close();
                    }
                }
 
                return stringLiteral;
            case NULL:
                lexer.nextToken();
                return null;
            case UNDEFINED:
                lexer.nextToken();
                return null;
            case TRUE:
                lexer.nextToken();
                return Boolean.TRUE;
            case FALSE:
                lexer.nextToken();
                return Boolean.FALSE;
            ...ʡ��...
}
```
����jackson�ķ�ʽ�����ݲ�ͬ����������ͬ�����ݴ���ͬ��2.00Ҳ����Ϊ��float���ͣ�ͬ����Ҫ����Ƿ���Feature.UseBigDecimal���ԣ�ֻ����FastJsonĬ�Ͽ����˴˹��ܣ�

###2.Protostuff����
����������һ����Json�����л���ʽ����protostuff����������������ģ�
׼�����Դ������£�

```
@SuppressWarnings("unchecked")
public class PBTest {
 
    public static void main(String[] args) {
        Bean1 bean1 = new Bean1("haha1", new BigDecimal("1.00"));
        Bean2 bean2 = new Bean2("haha2", new BigDecimal("2.00"));
 
        LinkedBuffer buffer1 = LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE);
        Schema schema1 = RuntimeSchema.createFrom(bean1.getClass());
        byte[] bytes1 = ProtostuffIOUtil.toByteArray(bean1, schema1, buffer1);
 
        Bean1 bean11 = new Bean1();
        ProtostuffIOUtil.mergeFrom(bytes1, bean11, schema1);
        System.out.println(bean11.toString());
 
        LinkedBuffer buffer2 = LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE);
        Schema schema2 = RuntimeSchema.createFrom(bean2.getClass());
        byte[] bytes2 = ProtostuffIOUtil.toByteArray(bean2, schema2, buffer2);
 
        Bean2 bean22 = new Bean2();
        ProtostuffIOUtil.mergeFrom(bytes2, bean22, schema2);
        System.out.println(bean22.toString());
 
    }
}
```
������£�

```
Bean1 [p1=haha1, p2=1.00]
Bean2 [p1=haha2, p2=2.00]
```
���Է���ProtostuffҲ�����ڴ����⣬ԭ������ΪProtostuff�����л���ʱ��ͽ����͵���Ϣ����ڶ������У���ͬ�����͸����˲�ͬ�ı�ʶ��RuntimeFieldFactory�г������б�ʶ��

```
public abstract class RuntimeFieldFactory<V> implements Delegate<V>
{
 
    static final int ID_BOOL = 1, ID_BYTE = 2, ID_CHAR = 3, ID_SHORT = 4,
            ID_INT32 = 5, ID_INT64 = 6, ID_FLOAT = 7,
            ID_DOUBLE = 8,
            ID_STRING = 9,
            ID_BYTES = 10,
            ID_BYTE_ARRAY = 11,
            ID_BIGDECIMAL = 12,
            ID_BIGINTEGER = 13,
            ID_DATE = 14,
            ID_ARRAY = 15, // 1-15 is encoded as 1 byte on protobuf and
            // protostuff format
            ID_OBJECT = 16, ID_ARRAY_MAPPED = 17, ID_CLASS = 18,
            ID_CLASS_MAPPED = 19, ID_CLASS_ARRAY = 20,
            ID_CLASS_ARRAY_MAPPED = 21,
 
            ID_ENUM_SET = 22, ID_ENUM_MAP = 23, ID_ENUM = 24,
            ID_COLLECTION = 25, ID_MAP = 26,
 
            ID_POLYMORPHIC_COLLECTION = 28, ID_POLYMORPHIC_MAP = 29,
            ID_DELEGATE = 30,
 
            ID_ARRAY_DELEGATE = 32, ID_ARRAY_SCALAR = 33, ID_ARRAY_ENUM = 34,
            ID_ARRAY_POJO = 35,
 
            ID_THROWABLE = 52,
 
            // pojo fields limited to 126 if not explicitly using @Tag
            // annotations
            ID_POJO = 127;
            ......
}
```
���л���ʱ���������¸�ʽ���洢���ݵģ�����ͼ��ʾ��
![ͼƬ����][1]

tag����������ֶε�λ�ñ�ʶ�������һ���ֶΣ��ڶ����ֶΡ����Լ�������Ϣ�����Կ�һ������bean���л�֮��Ķ�������Ϣ��

```
10 5 104 97 104 97 49 18 4 49 46 48 48
10 5 104 97 104 97 50 19 98 4 50 46 48 48 20
```
104 97 104 97 49��104 97 104 97 50�ֱ��ǣ�haha1��haha2��49 46 48 48��50 46 48 48�ֱ���1.00��2.00��
Bean2�洢����������ϸ��Bean1����ΪBean2�е�p2��ΪObject�洢����Ҫ�洢Object����ʼ��ʶ�ͽ�����ʶ������Ҫ��������������Ϣ��

������Բο���[https://my.oschina.net/OutOfMemory/blog/800226][2]

##�ܽ�
��Json���л���ʽ����û�б������ݵ����ͣ������ڷ�����ʱ��Щ���Ͳ������֣�ֻ��ͨ���������Եķ�ʽ�����������json��ʽ�и��õĿɶ��ԣ�ֱ�����л�Ϊ�����Ƶķ�ʽ�ɶ��Բ�㣬���ǿ��Խ��ܶ���Ϣ�����ȥ���������ƣ�

##ʾ�������ַ
[https://github.com/ksfzhaohui/blog][3]
[https://gitee.com/OutOfMemory/blog][4]


  [1]: /img/bVbivgp
  [2]: https://my.oschina.net/OutOfMemory/blog/800226
  [3]: https://github.com/ksfzhaohui/blog
  [4]: https://gitee.com/OutOfMemory/blog