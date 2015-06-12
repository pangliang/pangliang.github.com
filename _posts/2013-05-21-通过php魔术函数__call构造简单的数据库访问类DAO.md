---
layout: post
categories : [lnmp]
tags : []
excerpt: 缓存的使用比较普遍了, 如果构造一个简单的数据库访问类, 缩小业务代码的逻辑,
---
{% include JB/setup %}

{% highlight php linenos %}
<?php
class DAO
{
	private $sqls=array(
			'getActionMoSucc'=>'select * from xxxx where action=$0',
			);

	function __call($method,$args)
	{
		var_dump($method);
		var_dump($args);
		$key=substr($method,3).implode("",$args);
		var_dump($key);
		//if($value=$redis->get($key))
		//      return $value;
		//else
		$this->getFromMysql($method,$args);
	}
	function getFromMysql($key,$args)
	{
		$sql=$this->sqls[$key];
		foreach($args as $k=>$v)
			$sql=str_replace("\$$k",$v,$sql);
		var_dump($sql);
	}
}

$counter=new DAO();

$action="TEST";
$counter->getActionMoSucc($action);
?>
{% endhighlight %}


