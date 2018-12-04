## 1、直接拼SQL，并且不转义变量。（*此代码不能通过CodeReview*）

```
class OrderController {
    public function postGoods(Request $req)
        $name = $req->input('name');
        $sql = "SELECT * FROM `goods` WHERE `name` = '$name'";
    }
}
```

* 结论：容易导致SQL注入，$name = "' OR '1";时，将查询所有的商品。
* 如果用在登录上，将出现随便写个密码都能登录成功。
* 如果数据库支持多SQL，被攻击后，会导致删库删表的严重Bug。
* 解决方案：使用Lavaral的Model生成的SQL。

## 2、滥用Model的$appends（*此代码不能通过CodeReview*）

```
class Goods extends Model
{
    protected $appends = ['images', 'img_urls', 'category_name', 'category_id'];

    public function getImgUrlsAttribute() {
        $data = array();
        foreach ($this->images as $value)
            $data[] = cloud_file_url($value);

        return $data;
    }

    public function getImagesAttribute() {
        return $this->imgs()->select(['url'])->get()->pluck('url')->toArray();
    }

    public function getCategoryIdAttribute() {
        return $this->goodsCategory()->get()->pluck('id')->toArray();
    }

    public function getCategoryNameAttribute() {
        return $this->goodsCategory()->get()->pluck('name')->toArray();
    }
}
```

* 结论：查询图片的SQL（$this->imgs()->select(['url'])->get()）和查询分类的SQL（$this->goodsCategory()->get()），都被重复执行。
* 特别是商品列表页，列表中10个商品，每个都多余执行2次SQL，SQL被多余执行了20次。
* 解决方案：$appends中去掉多余的（如下）。
$appends只获取所有分类的信息（包括id和name），再从中找出id和name。

```
class Goods extends Model
{
    protected $appends = ['images', 'category'];

    public function getImagesAttribute() {
        return $this->imgs()->select(['url'])->get()->pluck('url')->toArray();
    }

    public function getCategoryAttribute() {
        return $this->goodsCategory()->get()->toArray();
    }
}
```

## 3、查询没用上已有的索引（*此代码不能通过CodeReview*）

```
CREATE TABLE `store_goods` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `store_id` int(11) NOT NULL DEFAULT '0' COMMENT '门店Id',
  `goods_id` int(11) NOT NULL DEFAULT '0' COMMENT '商品Id',
  PRIMARY KEY (`id`),
  KEY `idx_store_id_goods_id` (`store_id`,`goods_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='门店商品表';

查询SQL：SELECT `id`, `store_id`, `goods_id` FROM `store_goods` WHERE `goods_id` = 100
```
* 结论：只通过`goods_id`查询，用不到`idx_store_id_goods_id`索引。
* 解决方案1：SQL中加上store_id，如：SELECT `id`, `store_id`, `goods_id` FROM `store_goods` WHERE `store_id` = 200 AND `goods_id` = 100。
* 解决方案2：再加一个索引，KEY `idx_goods_id` (`goods_id`)，注：`idx_store_id_goods_id`索引不用删。
* 注：SELECT `id`, `store_id`, `goods_id` FROM `store_goods` WHERE `store_id` = 200，这条SQL能用上`idx_store_id_goods_id`索引。

## 4、不必要的SQL重复执行（*此代码不能通过CodeReview*）

```
$accept_name = $order->orderAddress()->first()['accept_name'] ?? '';
$mobile      = $order->orderAddress()->first()['mobile'] ?? '';
```
* 结论：$order->orderAddress()->first()中的SQL被执行2次。

## 5、SQL中使用Union All（*此代码不能通过CodeReview*）

```
$t1 = \DB::table('table1')->select(['id','created_at'])->where('id', 1);
$t2 = \DB::table('table2')->select(['id','created_at'])->where('id', 1)->unionAll($t1);
$sql = $t2->toSql();
```
* 解决方案：用2条SQL

## 6、SQL中出现LIKE，并且有前置%

```
WHERE nickname LIKE %$nickname% OR mobile LIKE %$mobile% OR ... LIMIT ...
```
* 结论：会全表遍历，效率很低
* 解决方案：Elasticsearch + Hbase

## 7、==和===优化

* ==和===，当确定条件表达式两边数据类型相同时，建议使用 ===
* 原因：
* 1)使用==，PHP会多做一次类型转换
* 2)特殊场景下会出bug，如：
```
// 0e开头，表示“科学计数法”
$a = '0e462097431906509019562988736854';
$b = '0e830400451993494058024219903391';

var_dump($a == $b); // true
var_dump($a === $b); // false
```
* 3)正常业务中可能会因为此Bug，被攻击。攻击示例如下：
* 正确的密码 == 'QNKCDZO'
* POST请求URL：domain.com/login.php?pwd=240610708
```
// md5('240610708') == 0e462097431906509019562988736854
// md5('QNKCDZO')   == 0e830400451993494058024219903391

if (md5($_POST['pwd']) == md5('QNKCDZO')) {
    此处会通过验证，造成Bug，用 === 才能避免Bug
}
```

## 8、''和""，当字符串中无变量时，建议''


## 9、防御式编程

```
class Foo {
    // 开发者：张三
    public static function fun1() {
        return ['a' => 123];
    }
}

class Bar {
    // 开发者：李四
    public static function fun2() {
        $data = Foo::fun1();
        $a = $data['a'];
    }
}
```
* Bar::fun2中使用Foo::fun1的结果，但没有验证$data是否是数组，Key-a是否存在。
* 正确的做法是 is_array($data) && isset($data['a']) 后，才取$data['a']的值。
* 原因：如果张三突然改了Foo::fun1()，返回值可能是null，也可能少了Key-a，他通常不会通知李四去修改Bar::fun2()的。此时Bar::fun2()就报致命错误了。

## 10、除法运算，除数判0

$r = $a / $b; 运算前，确保$b !== 0
