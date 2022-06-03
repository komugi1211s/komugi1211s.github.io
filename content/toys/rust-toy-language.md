---
title: "Rust toy language"
date: 2022-04-20T11:15:24+09:00
draft: false
---

## About
[自作言語開発の為の入門チュートリアル](http://craftinginterpreters.com/)をネットで見つけ、面白そうだなと思ったので読みながらチマチマやっていたもの。

次第に自己流の型推論とか似ても似つかない構文とか自分が付けたい仕様を追加しようと四苦八苦していたら取り返しがつかない道の外し方をしていた。


## Codes

~~~
simple_fibonacci: fn(number: int) int {
    if number == 0 return 0;
    if number == 1 return 1;
    return simple_fibonacci(number - 1) + simple_fibonacci(number - 2);
}

print simple_fibonacci(15); // print 610
~~~

~~~
simple_fizz_buzz: fn(num: int) {
    i := 1;
    while (i < num) {
        if      i % 15 == 0 print "FizzBuzz";
        else if i %  5 == 0 print "Buzz";
        else if i %  3 == 0 print "Fizz";
        else                print i;

        i = i + 1;
    }
}

simple_fizz_buzz(100);
~~~
