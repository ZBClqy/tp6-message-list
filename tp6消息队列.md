## thinkphp做秒杀

做秒杀首先我们就要知道mq也就是我们的消息队列，这是现在一般流行也是我们传统采用的方式

首先我们需要安装一个关于thinkphp的扩展也就是我们的think-queue

### thinkphp-queue

这是官网提供的一个消息队列服务，他支持我们的消息队列的常规操作

消息的发布 获取 执行 删除 重发 失败处理 延迟执行 超时控制等

队列的多队列 内存限制 启动 停止 和守护等

消息队列可以降级为同步执行

composer require topthink/think-queue

然后配置我们config的queue.php文件

return [    'default'     => 'redis',    'connections' => [        'sync'     => [            'type' => 'sync',        ],        'database' => [            'type'       => 'database',            'queue'      => 'default',            'table'      => 'jobs',            'connection' => null,        ],        'redis'    => [            'type'       => 'redis',            'queue'      => 'default',            'host'       => env('redis.host', '127.0.0.1'),            'port'       => env('redis.port', 6379),            'password'   => env('redis.password', ''),            'select'     => 0,            'timeout'    => 0,            'persistent' => false,        ],    ],    'failed'      => [        'type'  => 'none',        'table' => 'failed_jobs',    ], ];

### 具体的使用

首先我们需要创建一个生产者

class Index extends BaseCotroller

{

​	public function index(){

​		$jobHandler='这里的路径是你处理队列的类比如  app\api\controller\job1';

​		$jobQueueName='helloJonQueue'; //这里存放的是当前redis队列的名称，如果是新队列，会自动的创建

​		$jobData=['ts'=>time(),'bizId'=>uniqId(),'a'=>1];//这里模拟一个我们要传给队列处理的消息

​		$isPushed=Queue::later(10,$jobHandler,$jobData,$jobQueueName)//这里将我们的消息压入队列，这里我用的是later延时执行，如果想马上执行的话要使用我们的push

​		//最后这里会返回我们的在队列中的下标

​		if($isPushed!==false){

​				//存在下标则成功

​				echo '推送成功'

​		}else{

​				//不存在则失败

​				echo '推送失败'

​		}

​	}

}

有了我们的生产者也就需要我们的消费者也就是我们队列的执行类 方法

class Job1

{

​		public function fire(Job $job,array $data){

​			$isJobData=$this->checkData($data);

​			if(!$isJobData){

​				$job->delete();

​				return;

​			}

​			$isJobDone = $this->doHelloJob($data);

​			if($isJobDone){

​				$job->delete();

​				echo "执行完删除队列中的当前任务";

​			}else{

​				if($job->attempts()>3){

​					$job->delete();

​					echo "当我们的任务超时也将其删除"

​				}

​			}

​		}

​		//我们使用下面这个方法去过滤一下我们队列传进来的数据，如果不是我们想要的就将其过滤掉

​		private function checkData(array $data){

​			return true

​		}

​		//根据下面的方法进行实际业务的处理

​		private function doHelloJob(array $data){

​			echo '执行业务逻辑:'.$data['bizId'].'\n';

​			return true;

​		}

}

一切准备好后就可以启动我们的php程序了

然后最后我们在开启一个我们的守护进程即可

php think queue:work --queue helloJobQueue

守护进程这个进程就是用来处理我们的队列中消息的因为我们队列中的消息是用我们的命令行执行的所以就需要多开一个守护进程去帮我们执行