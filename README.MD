# 分布式延迟调度中间件（delay-scheduler）
自实现Raft分布式一致性算法+LevelDB实现分布式action日记存储，使用时间轮算法调度action，支持取消action。
## 流程
* 只有一个Leader节点处理客户端的请求；
* 客户端向Leader提交action时，action被封装为日记同步到其它节点，存储到Leveldb；
* Leader节点负责扫描本机存储状态为pending的action放入时间轮，放入成功只更新节点自身存储的action状态为调度中；
* 时间轮算法将到时间执行的action放入一个阻塞队列；
* 客户端主动请求Leader节点从阻塞队列拉取时间轮中的action执行；
* 客户端执行完成action后提交结果，Leader节点与其它节点都根据消费结果更新action状态；
* 如果action状态为消费失败，则刷新提交时间为当前时间，并且计算下一次执行的相对延时，更新状态为pending；
## 定时action
* 只能指定一个执行时间，且不管成功失败都只调度一次；
## 延迟action
* 支持自定义延迟重试周期，如(2,4,6,8,10（单位秒）)，最少1个，最大256个
* 1、如果需要立即执行一次，则应该这样配置：0,2,4,...
* 2、当客户端提交任务执行成功时，将不会再进行下一次的延迟重试调度；
## 取消action
* action状态为pending时允许取消；
* 如果当前时间已经达到action的首次调度时间，将不允许取消；
## Q&A
* 1、当Leader挂掉时，已经提交到时间轮中的action或者已经被客户端拉取但未提交消费成功的action，
都会被新的节点重新调度；（因此需要使用者自行确保幂等消费）
* 2、客户端已经拉取消费，但未消费或者消费完成但还未提交就已经停止进程了，那么该action在当前Leader节点会一直处于锁定状态，
一段时间后，如果当前节点还是主节点，则恢复action状态为pending（其它节点成为Leader会重新调度该action，也会同步状态给当前action，
如果没调度就又轮回当前节点为Leader，那么一段时间后由自身恢复）。
* 3、action不一定准时执行，当Leader挂掉时、或者没有客户端拉取，那么action都会被延迟执行。