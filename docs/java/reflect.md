## 1、`反射`

1. `简介`：JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。要想解剖一个类,必须先要获取到该类的字节码文件对象。而解剖使用的就是Class类中的方法.所以先要获取到每一个字节码文件对应的Class类型的对象.
2. `获取Class对象方式`：Object.getClass()、通过Class静态方法forName
3. `Spring`使用频率很高
4. `原理`：每个类都对应Class对象，在面向对象世界，类也是对象。那么我们通过拿到Class对象，就相当于拿到了方法区中对象信息了，那么可以通过这个类类型对象拿到类中的东西了。
