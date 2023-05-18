#### 11.1.7　Redis
この章ではRedisの並行処理の側面に注目する。
Redisリストを使えば、キューは手っ取り早く作れる。Redisサーバーは1台のマシンで実行され、このマシンはクライアントと同じものでもクライアントがネットワークを介してアクセスできる別のマシンでも良い。いずれにしても、クライアントはＴＣＰを介してサーバーとやり取りするため、両者はネットワークでつながっている。  
ひとつ以上のワーカークライアントがブロックを起こすポップ処理でこのリストを監視している。リストが空なら、ワーカークライアントは動かないが、メッセージが届くと、そのなかのどれかがメッセージを取りに行く。
redis_washer.pyとredis_dryer.pyを例として動作を見る。

~~~python
#redis_washer.py
import redis
conn = redis.Redis()
print('Washer is starting')
dishes = ['salad', 'bread', 'entree', 'dessert']
#皿の名前が書かれている４つのメッセージを生成
for dish in dishes:
    msg = dish.encode('utf-8')
    #Redisサーバー内のdishesというリストにメッセージを追加
    conn.rpush('dishes', msg)
    print('Washed', dish)
conn.rpush('dishes', 'quit') #最後にquitを送る
print('Washer is done')
~~~
~~~python
#redis_dryer.py
import redis
conn = redis.Redis()
print('Dryer is starting')
while True:
    msg = conn.blpop('dishes')  #dishesになっているメッセージを待つ
    if not msg:
        break
    val = msg[1].decode('utf-8')
    if val == 'quit':  #quitが届いたらループを終了
        break
    print('Dride', val) 
print('Dishes are dried')
~~~
乾燥担当を起動してから洗浄担当を起動する。  
コマンドラインで最後に&を追加しているので、第１のプログラムはバックグラウンドで実行されるため、実行され続けているがキーボードに反応しなくなる。

~~~
$ python3 redis_dryer.py &
[1] 2517
$ Dryer is starting
python3 redis_washer.py
Washer is starting
Washed salad
Dride salad
Washed bread
Dride bread
Washed entree
Dride entree
Washed dessert
Dride dessert
Washer is done
Dishes are dried
[1]+  Done                    python3 redis_dryer.py
~~~


redis_dryer2.py
def dryer():
    import redis
    import os
    import time
    conn = redis.Redis()
    pid = os.getpid()
    timeout = 20
    print('Dryer process %s is starting' % pid)
    while True:
        msg = conn.blpop('dishes', timeout)
        if not msg:
            break
        val = msg[1].decode('utf-8')
        if val == 'quit':
            break
        print('%s: dried %s' % (pid, val))
        time.sleep(0.1)
    print('Dryer process %s is done' % pid)

import multiprocessing
DRYERS=3
for num in range(DRYERS):
    p = multiprocessing.Process(target=dryer)
    p.start()
    
    
