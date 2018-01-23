# Метрические алгоритмы классификации
## Алгоритм k ближайших соседей (kNN)
- **KNN** - метрический алгоритм классификации, основанный на оценивании сходства объектов. Классифицируемый объект относится к тому классу, которому принадлежат ближайшие к нему объекты обучающей выборки. 

Данный алгоритм, как и пара следующих рассмотренных методов, основываются на _**гипотезе компактности:**_ если мера сходства объектов введена достаточно удачно, то схожие объекты гораздо чаще лежат в одном классе, чем в разных. В этом случае граница между классами имеет достаточно простую форму, а классы образуют компактно локализованные области в пространстве объектов. 

Пусть задана обучающая выборка пар «объект-ответ».![](http://latex.codecogs.com/gif.latex?X%5Em%3D%5C%7B%28x_1%2Cy_1%29%2C%5Cdots%2C%28x_m%2Cy_m%29%5C%7D)

Пусть на множестве объектов задана функция расстояния ![](http://latex.codecogs.com/gif.latex?%5Crho%28x%2Cx%27%29). Эта функция должна быть достаточно точной _"мерой"_ сходства объектов.
Для точки **_u_** выборки отсортируем остальные её объекты по возрастанию значения расстояния до **_u_**.
Для успешного обучения выборка должна быть пересортирована для каждого новой точки **_u_**. 

В общем виде алгоритм **_kNN_** выглядит так :
![](http://latex.codecogs.com/gif.latex?a%28u%29%3D%5Cmathrm%7Barg%7D%5Cmax_%7By%5CinY%7D%5Csum_%7Bi%3D1%7D%5Em%5Cbigl%5Bx_%7Bi%3Bu%7D%3Dy%5Cbigr%5Dw%28i%2Cu%29),
где ![](http://latex.codecogs.com/gif.latex?w%28i%2Cu%29) — мера _«важности»_ (вес) объекта ![](http://latex.codecogs.com/gif.latex?x_u%5E%7Bi%7D)

Алгоритм зависит от параметра _k_, оптимальное значение которого определяется по критерию скользящего контроля,в нашем случае используется метод исключения объектов по одному (leave-one-out cross-validation).

**_Проще говоря_**, алгоритм от произвольной точки _U_ сортирует остальную выборку по расстоянию до _U_, и _относит её к классу_, который имеет максимальное значение объектов среди первых **_K_** соседей **U**.

### Реализация

Src: [ссылка](kNN.R)  
Основной интерес реализации заключён в двух функциях:

### KNN
```R
#Функция получает в качестве параметров отсортированную выборку и количество ближайших соседей
DT.kNN.kNN = function(sortedDist, k) { 
    kDist = sortedDist[1:k] #Берём только первых K отсортированных объекта выборки
    kClasses = names(kDist) #получаем названия классов первых K соседей с помощью функции names()
    class = names(which.max(table(kClasses))) 
    #функция возвращает название класса, объектов которого больше всего среди K соседей.
    return(class)
}
```
### LOO
Данная функция была обобщена для трёх классификаторов и была вынесена в отдельный исходник,функция принимает points - массив точек обучающей выборки, в нашем случае - столбцы матрицы iris[,3:4];  
classes - столбец с названиями классов элементов выборки, iris[,5]  
Для определения классификатора и выбора соответствующего варианта подсчёта LOO передаётся функция классификатора; hlim - параметр для Парзеновского окна, для kNN не имеет значения
```R
DT.Util.LOO = function(points, classes, classFunc, hLims = 0) {
  n = dim(points)[1]# берём количество элементов из первого столбца
  if(identical(classFunc, DT.kNN.kNN) || identical(classFunc, DT.WkNN.WkNN)) { #сравнение методов,см выше
  loo = double(n-1) #n-1 поскольку мы всегда удаляем один элемент для LOO
  
  for (i in 1:n) { #идём по выборке
  #находим эвклидовы расстояния от нашей точки до оставшейся выборки без нашей точки - points[-i]
    distances = DT.Util.getDist(points[-i,], points[i,], DT.Util.euclidDist) 
    names(distances) = classes[-i] #присваиваем имена классов точек в массиве расстояния
    sortDist = sort(distances) #сортируем по расстоянию
    #вложенный цикл нужен для подсчёта 
    for (l in 1:n - 1) { 
      bestClass = classFunc(sortDist, l) #вызов kNN для l соседей
      loo[l] = loo[l] + ifelse(bestClass == classes[i], 0, 1) 
      } #если классификатор ошибся, увеличиваем LOO при l соседях
    }
  } # вот так вот просто можно поделить весь массив на число элементов выборки
    loo = loo / n
  return(loo)
}
```


![](pics/kNN.png)

Используется датасет iris по лепесткам (ширина и длина -наиболее подходящие для классификации параметры )

Подбор оптимального количества соседней (_k_) отображается на графике **LOO** слева. Для реализованного классификатора _k_ = 6, при **_LOO_** равное 0.033, что соответствует ~5 неправильно классифицируемым элементам выборки.

Параметр _k_, как видно на графике LOO, в первой сотне имеет относительно низкий показатель ошибок классификации, при больших показателях _k_ погрешность стремительно растёт. Происходит это из - за фундаментального недостатка самого алгоритма **KNN**, который **не**
учитывает расстояние точки до соседей, учитывая <u>только</u> их наличие (Исправлено в ***kWNN***). Т.е, у **_n_**-ого соседа будет **"вес"** как у, например, ближайшего.  

----

# Байесовские алгоритмы классификации
Байесовские алгоритмы классификации основаны на принципе максимума апостериорной вероятности : для классифицируемого объекта вычисляются плотности распределения ![](http://latex.codecogs.com/svg.latex?%5Cinline%20p%28x%7Cy%29%20%3D%20p_y%28x%29)  — **_функции правдоподобия_** классов, по ним вычисляются ***апостериорные вероятности*** - ![](http://latex.codecogs.com/svg.latex?P%20%5Cleft%20%5C%7By%7Cx%20%5Cright%20%5C%7D%20%3D%20P_yp_y%28x%29), где ![](http://latex.codecogs.com/svg.latex?%5Cinline%20P_y)- ***априорные вероятности*** классов. Объект относится к классу с максимальной апостериорной вероятностью.

*Задача классификации* - получить алгоритм ![](http://latex.codecogs.com/svg.latex?%5Cinline%20a%3A%5C%3B%20X%5Cto%20Y), способный классифицировать произвольный объект ![](http://latex.codecogs.com/svg.latex?%5Cinline%20x%20%5Cin%20X).  

1)  ***Построение классификатора при известных плотностях***  
![](http://latex.codecogs.com/svg.latex?%5Cinline%20%5Clambda_y) - штраф за неправильное отнесение объекта класса ***𝑦***.  
Если известны ![](http://latex.codecogs.com/svg.latex?%5Cinline%20P_y)  и ![](http://latex.codecogs.com/svg.latex?%5Cinline%20p_%7By%7D%28x%29), то минимум среднего риска ![](http://latex.codecogs.com/svg.latex?%5Cinline%20R%28a%29%20%3D%20%5Csum_%7By%5Cepsilon%20Y%7D%20%5Csum_%7Bs%5Cepsilon%20Y%7D%20%5Clambda_yP_yP%28A_s%7Cy%29), ![](http://latex.codecogs.com/svg.latex?%5Cinline%20A_s%20%3D%20%5Cbigl%5C%7Bx%20%5Cin%20X%7Ca%28x%29%3Ds%5Cbigr%5C%7D%2C)  достигается алгоритмом ![](http://latex.codecogs.com/svg.latex?%5Cinline%20a%28x%29%20%3D%20%5Carg%5Cmax%20%5Clambda_yP_yp_y%28x%29)

2) ***Восстановление плотностей по выборке***  
По подвыборке  класса *y* строим эмпирические оценки  ![](http://latex.codecogs.com/svg.latex?%5Cinline%20P_y) (доля объектов в выборке) и ![](http://latex.codecogs.com/svg.latex?%5Cinline%20p_y%28x%29).  
Три метода:  
**1)Параметрический** если плотности нормальные (гауссовские) - НДА и ЛДФ;  
**2)Непараметрический** - оценка Парзена - Розенблатта, метод парзеновского окна;   
**3)Разделение смеси** производится _ЕМ-алгоритмом_. Плотности компонент смеси (гауссовские плотности) - радиальные функции,метод радиальных базисных функций.


***Линейный дискриминант Фишера***

***Ковариационные матрицы*** классов равны, классы **s, t** равновероятны и равнозначны ![](https://latex.codecogs.com/gif.latex?%5Clambda_sP_s%20%3D%20%5Clambda_tP_t), признаки некоррелированы и имеют одинаковые ***дисперсии*** ![](https://latex.codecogs.com/gif.latex?%5Csum_s%20%3D%20%5Csum_t%20%3D%20%5Csigma%20I_n).  

Это означает, что классы имеют одинаковую сферическую форму, разделяющая плоскость проходит посередине между классами, ортогонально линии, соединяющей центры классов. Нормаль оптимальна - прямая, в одномерной проекции на которую классы разделяются наилучшим образом,с наименьшим байесовским риском **R(a)**.

<img src="pics/FLDe.png" height = "200" width="500">

Применяя ![](https://latex.codecogs.com/gif.latex?%5Cln%20p_y%28x%29%20%3D%20-%5Cfrac%7Bn%7D%7B2%7D%5Cln2%5Cpi%20-%20%5Cfrac%7B1%7D%7B2%7D%20%5Cln%20%7C%5Csum_y%7C-%5Cfrac%7B1%7D%7B2%7D%28x%20-%20%5Cmu_y%29%5ET%5Csum%5E%7B-1%7D_y%28x-%5Cmu_y%29) , квадратичные члены сокращаются и уравнение поверхности
вырождается в линейную форму: ![](https://latex.codecogs.com/gif.latex?%28x-%5Cmu_%7Bst%7D%29%5ET%5Csum%5E%7B-1%7D%28%5Cmu_s-%5Cmu_t%29%20%3D%20C_%7Bst%7D), где ![](https://latex.codecogs.com/gif.latex?%5Cmu_%7Bst%7D%20%3D%20%5Cfrac%7B1%7D%7B2%7D%28%5Cmu_s&plus;%5Cmu_t%29) - точка посередине между центрами классов.


### Реализация

Src: [ссылка](LDF.R) 

Код существенно не отличается от предыдущего алгоритма.

**Результаты**

<img src="pics/FLD1.png" width="500">

Алгоритм неплохо работает, когда формы классов действительно близки к нормальным и не слишком сильно различаются.  

В этом случае линейное решающее правило близко к оптимальному байесовскому, но устойчивее квадратичного, и часто обладает лучшей обобщающей способностью.

-------

# Линейные алгоритмы классификации 


Рассматривается задача классификации с двумя классами **Y={-1,+1}**. Модель алгоритма - параметрическое отображение ***a(x,w) = sign f(x, w)***  , где ***w - вектор параметров***, а ***f(x, w)*** - *дискриминантная функция*. 

Если её значение **> 0**, объект относится к классу **+1**, иначе **-1**. Уравнение ***f(x, w) = 0*** описывает разделяющую поверхность. 

![](http://latex.codecogs.com/svg.latex?M_i%28w%29%20%3D%20y_if%28x_i%2Cw%29) - *отступ* объекта ![](http://latex.codecogs.com/svg.latex?%5Cinline%20x_i) относительно классификатора. Если отступ отрицательный - алгоритм ошибся в классификации на объекте. Больше отступ - правильнее и надёжнее классификация объекта ![](http://latex.codecogs.com/svg.latex?%5Cinline%20x_i).  


![](http://latex.codecogs.com/svg.latex?%5Cinline%20%5Calpha%20%28M_i%28w%29%29) - *функция потерь*, а функция отступа - монотонно невозрастающая, мажорирующая пороговую ф-ю потерь : ![](http://latex.codecogs.com/svg.latex?%5Cinline%20%5BM%3C0%5D%5Cleqslant%20%5Calpha%20%28M%29). Тогда минимизация суммарных потерь - это метод минимизации числа ошибок на выборке: 

![](http://latex.codecogs.com/svg.latex?Q%28w%2CX%5El%29%20%3D%20%5Csum_%7Bi%3D1%7D%5E%7Bl%7D%5BM_i%28w%29%20%3C0%5D%20%5Cleqslant%20%5Coverset%7B-%7D%7BQ%7D%28w%2CX%5El%29%20%3D%20%5Csum_%7Bi%3D1%7D%5El%20%5Calpha%28M_i%28w%29%29%20%5Crightarrow%20%5Cunderset%7Bw%7D%7B%5Cmin%7D) 


Если дискриминантная функция - ![](http://latex.codecogs.com/svg.latex?%5Cinline%20%5Cleft%20%5Clangle%20x%2Cw%20%5Cright%20%5Crangle%2C%20w%5Cin%20%5Cmathbb%7BR%7D%5En), получим ***линейный классификатор***:  ![](http://latex.codecogs.com/svg.latex?a%28x%2Cw%29%20%3D%20%5Cmathrm%7Bsign%7D%28%5Cleft%20%5Clangle%20w%2Cx%20%5Cright%20%5Crangle%20-%20w_0%29%20%3D%20%5Cmathrm%7Bsign%7D%28%5Csum_%7Bj%3D1%7D%5Enw_jf_j%28x%29-w_0%29)

***Стохастический градиент***  

Необходимо найти вектор параметров ![](http://latex.codecogs.com/svg.latex?w%20%5Cin%20%5Cmathbb%7BR%7D%5En), где достигается минимум эмпирического риска.

Веса _w_  подбираются в цикле, на каждом шаге веса сдвигаются в направлении антиградиента  
![](http://latex.codecogs.com/svg.latex?Q%27%28w%29%20%3D%20%5CBigr%28%5Cfrac%7B%5Cpartial%20Q%28w%29%7D%7B%5Cpartial%20w_j%7D%5CBigr%29%5En_%7Bj%3D1%7D)  

Алгоритм получает обучающую выборку, темп обучения ![](http://latex.codecogs.com/svg.latex?%5Ceta) и параметр сглаживания ![](http://latex.codecogs.com/svg.latex?%5Clambda). Перед применением метода выборка подготавливается и нормируется:  

__Признаковое нормирование__

![](http://latex.codecogs.com/svg.latex?f_j%20%3D%20%5Cfrac%7Bf_j%20-%20m%7D%7B%5Csigma%7D)
, где _m_ – среднее арифмитическое значение признака _j_,
![](http://latex.codecogs.com/svg.latex?%5Csigma)
– среднеквадратическое отклонение.

__Подготовка__

Разделяющая поверхность
![](http://latex.codecogs.com/svg.latex?%5Clangle%20w%2C%20x%20%5Crangle%20%3D%200).

У нас объект имеет всего два признака.
![](http://latex.codecogs.com/svg.latex?w_1x_1%20&plus;%20w_2x_2%20%3D%200).
У разделяющей прямой нет св. коэфф-та, добавим фиктивный параметр *=-1* :
![](http://latex.codecogs.com/svg.latex?w_1x_1%20&plus;%20w_2x_2%20-%20w_3%20%3D%200).

***Подробный алгоритм SG***

1. Инициализация весов
![](http://latex.codecogs.com/svg.latex?w_j%2C%20j%3D1%2C...%2Cn).

2. Вычисление начального приближения
![](http://latex.codecogs.com/svg.latex?Q%20%3D%20%5Csum_%7Bi%20%3D%201%7D%5E%7B%5Cell%7D%20%5Cmathcal%7BL%7D%28%5Clangle%20w%2C%20x_i%20%5Crangle%20y_i%29)

3. ***Пока  _Q_ не стабилизировано*** и в выборке присутствуют объекты с отрицательным **М**, ***повторять:***  
Условия выше иногда может быть не достаточно, алгоритм может остановиться, не получив необходимого результата, если два раза подряд выберет похожие элементы. Увеличим количество повторов выбора до десяти во избежание этого (меньшие значения были также проверены, и их было недостаточно).
 
4. Выбрать случайный элемент ![](http://latex.codecogs.com/svg.latex?x_i) 
5. Ошибка: ![](http://latex.codecogs.com/svg.latex?%5Cvarepsilon%20_i%20%3D%20%5Cmathcal%7BL%7D%28%5Clangle%20w%2C%20x_i%20%5Crangle%20y_i%29)
6. Шаг градиентного спуска: ![](http://latex.codecogs.com/svg.latex?w%20%3D%20w%20-%20%5Ceta%20%5Cmathcal%7BL%7D%27%28%5Clangle%20w%2C%20x_i%20%5Crangle%20y_i%29x_iy_i)
7. Оценка: ![](http://latex.codecogs.com/svg.latex?Q%20%3D%20%281%20-%20%5Clambda%29Q%20&plus;%20%5Clambda%20%5Cvarepsilon_i)

Линейные алгоритмы отличаются функцией потерь
![](http://latex.codecogs.com/svg.latex?%5Cmathcal%7BL%7D%28%5Clangle%20w%2C%20x_i%20%5Crangle%20y_i%29)
, где
![](http://latex.codecogs.com/svg.latex?%5Clangle%20w%2C%20x_i%20%5Crangle%20y_i)
– отступ.

***ADALINE***

Линейный алгоритм классификации, основан на методе стохастического градиента
![](http://latex.codecogs.com/svg.latex?%5Cmathcal%7BL%7D%28M%29%20%3D%20%28M%20-%201%29%5E2%20%3D%20%28%5Clangle%20w%2Cx_i%20%5Crangle%20y_i%20-%201%29%5E2). - квадратичная функция потерь.
Производная берётся по _w_ и равна ![](http://latex.codecogs.com/svg.latex?%5Cmathcal%7BL%7D%27%28M%29%20%3D%202%28%5Clangle%20w%2Cx_i%20%5Crangle%20-%20y_i%29x_i).  
Получили правило обновления весов на каждой итерации метода *SG* - **дельта - правило:**
![](http://latex.codecogs.com/svg.latex?w%20%3D%20w%20-%20%5Ceta%28%5Clangle%20w%2Cx_i%20%5Crangle%20-%20y_i%29x_i).

 [Программная реализация](https://zoncker.shinyapps.io/LinearMerged/) была выполнена с использованием библиотеки Shiny(Бесплатный хост - это нечто!) для построения графического интерфейса (пусть и ужасного), стохастический градиент был унифицирован для трёх классификаторов, также на выбор пользователю предлагается два набора параметров задания выборки, когда она линейно - разделима и когда - нет. [Исходник](../LinearMerged/app.R)
 
 Результаты работы:
 
 ![](pics/ada0.png) ![](pics/ada1.png)
 
### Реализация

Src: [ссылка](ADALINE.R) )  
