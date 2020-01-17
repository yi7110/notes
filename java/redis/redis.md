# redis结合springboot项目的使用  
## redisDemo   
[redisDemo](https://blog.csdn.net/lms1719/article/details/83652578)

+ `pom.xml`  
```xml
 		<dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
        
<!-- https://mvnrepository.com/artifact/org.redisson/redisson -->
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson</artifactId>
            <version>3.10.5</version>
        </dependency>
```
+ `application.properties`  
```xml
# redis链接地址
spring.redis.host=192.168.222.20
# redis端口号
spring.redis.port=6379
# redis数据库
spring.redis.database=0
```
+  `RedisUtil`  
```java
public class RedisUtil {

    private JedisPool jedisPool;

    public void initPool(String host,int port ,int database){
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(200);
        poolConfig.setMaxIdle(30);
        poolConfig.setBlockWhenExhausted(true);
        poolConfig.setMaxWaitMillis(10*1000);
        poolConfig.setTestOnBorrow(true);
        jedisPool=new JedisPool(poolConfig,host,port,20*1000);
    }

    public Jedis getJedis(){
        Jedis jedis = jedisPool.getResource();
        return jedis;
    }

}
```
+ `RedisConfig`  
```java
//jedis配置
@Configuration
public class RedisConfig {
    //读取配置文件中的redis的ip地址
    @Value("${spring.redis.host:disabled}")
    private String host;
    @Value("${spring.redis.port:0}")
    private int port ;
    @Value("${spring.redis.database:0}")
    private int database;
    @Bean
    public RedisUtil getRedisUtil(){
        if(host.equals("disabled")){
            return null;
        }
        RedisUtil redisUtil=new RedisUtil();
        redisUtil.initPool(host,port,database);
        return redisUtil;
    }
}

//Redission配置
@Configuration
public class GmallRedissonConfig {

    @Value("${spring.redis.host:0}")
    private String host;

    @Value("${spring.redis.port:6379}")
    private String port;

    @Bean
    public RedissonClient redissonClient(){
        Config config = new Config();
        config.useSingleServer().setAddress("redis://"+host+":"+port);
        RedissonClient redisson = Redisson.create(config);
        return redisson;
    }
}
```
+ `Impl`  
```java
//jedis实现类
@Service
public class SkuServiceImpl implements SkuService {
    
    @Autowired
    RedisUtil redisUtil;

    public PmsSkuInfo getSkuByIdFromDb(String skuId){
        // sku商品对象
        PmsSkuInfo pmsSkuInfo = new PmsSkuInfo();
        pmsSkuInfo.setId(skuId);
        PmsSkuInfo skuInfo = pmsSkuInfoMapper.selectOne(pmsSkuInfo);

        // sku的图片集合
        PmsSkuImage pmsSkuImage = new PmsSkuImage();
        pmsSkuImage.setSkuId(skuId);
        List<PmsSkuImage> pmsSkuImages = pmsSkuImageMapper.select(pmsSkuImage);
        skuInfo.setSkuImageList(pmsSkuImages);
        return skuInfo;
    }

    @Override
    public PmsSkuInfo getSkuById(String skuId,String ip) {
        System.out.println("ip为"+ip+"的同学:"+Thread.currentThread().getName()+"进入的商品详情的请求");
        PmsSkuInfo pmsSkuInfo = new PmsSkuInfo();
        // 链接缓存
        Jedis jedis = redisUtil.getJedis();
        // 查询缓存
        String skuKey = "sku:"+skuId+":info";
        String skuJson = jedis.get(skuKey);

        if(StringUtils.isNotBlank(skuJson)){//if(skuJson!=null&&!skuJson.equals(""))
            System.out.println("ip为"+ip+"的学:"+Thread.currentThread().getName()+"从缓存中获取商品详情");

            pmsSkuInfo = JSON.parseObject(skuJson, PmsSkuInfo.class);
        }else{
            // 如果缓存中没有，查询mysql
            System.out.println("ip为"+ip+"的同学:"+Thread.currentThread().getName()+"发现缓存中没有，申请缓存的分布式锁："+"sku:" + skuId + ":lock");

            // 设置分布式锁
            String token = UUID.randomUUID().toString();
            String OK = jedis.set("sku:" + skuId + ":lock", token, "nx", "px", 10*1000);// 拿到锁的线程有10秒的过期时间
            if(StringUtils.isNotBlank(OK)&&OK.equals("OK")){
                // 设置成功，有权在10秒的过期时间内访问数据库
                System.out.println("ip为"+ip+"的同学:"+Thread.currentThread().getName()+"有权在10秒的过期时间内访问数据库："+"sku:" + skuId + ":lock");

                pmsSkuInfo =  getSkuByIdFromDb(skuId);

                if(pmsSkuInfo!=null){
                    // mysql查询结果存入redis
                    jedis.set("sku:"+skuId+":info",JSON.toJSONString(pmsSkuInfo));
                }else{
                    // 数据库中不存在该sku
                    // 为了防止缓存穿透将，null或者空字符串值设置给redis
                    jedis.setex("sku:"+skuId+":info",60*3,JSON.toJSONString(""));
                }

                // 在访问mysql后，将mysql的分布锁释放
                System.out.println("ip为"+ip+"的同学:"+Thread.currentThread().getName()+"使用完毕，将锁归还："+"sku:" + skuId + ":lock");
                String lockToken = jedis.get("sku:" + skuId + ":lock");
                if(StringUtils.isNotBlank(lockToken)&&lockToken.equals(token)){
                    //jedis.eval("lua");可与用lua脚本，在查询到key的同时删除该key，防止高并发下的意外的发生
                    jedis.del("sku:" + skuId + ":lock");// 用token确认删除的是自己的sku的锁
                }
            }else{
                // 设置失败，自旋（该线程在睡眠几秒后，重新尝试访问本方法）
                System.out.println("ip为"+ip+"的同学:"+Thread.currentThread().getName()+"没有拿到锁，开始自旋");

                return getSkuById(skuId,ip);
            }
        }
        jedis.close();
        return pmsSkuInfo;
    }
}
//redission实现类
@Controller
public class RedissonController {

    @Autowired
    RedisUtil redisUtil;

    @Autowired
    RedissonClient redissonClient;

    @RequestMapping("testRedisson")
    @ResponseBody
    public String testRedisson(){
        RLock lock = redissonClient.getLock("lock");// 声明锁
        lock.lock();//上锁
        try {
            // 设置字符串
            RBucket<String> keyObj = redissonClient.getBucket("k");
            String v = keyObj.get();
            if (StringUtils.isBlank(v)) {
                v = "1";
            }
            keyObj.set((Integer.parseInt(v) + 1) + "");
            System.out.println("->" + v);
        }finally {
            lock.unlock();// 解锁
        }
        return "success";
    }
}
```