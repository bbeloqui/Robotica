# Práctica Follow Line



En primer lugar para la realización de la práctica tenemos que obtener la imagen de nuestro campo de visón. Para ello utilizamos la sentencia que nos proporcionan "HAL.getImage()". Con esta sentencia obtenemos la imagen que vemos a continucaión.

![](https://github.com/bbeloqui/Robotica/blob/main/Follow_Line/vision_inicial.PNG)

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
### Filtro de color
 Una vez que ya tenemos la imagen lo, que nos interesa es quedarnos ( en forma binaria ) con la línea roja ya que es lo que tienemos que seguir. Para esto las imágenes que vamos recibiendo las convertimos a HSV, ya que los cambios de iluminación le afectan menos. Para extraer la línea vamos a dar valores máximos y mínimos a cada canal.
 - Canal H:  min = 0, max = 255
 - Canal S:  min = 77, max = 255
 - Canal V:  min = 56, max = 255
 
Luego, tenemos que unir los 3 canales para formar nuestra imagen y una vez que la tenemos, realizamos una binarización con la sentencia "threshold". Una vez realizado esto ya tenemos la línea aislada para poder tratarla.
 
````
    img_hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)
    line = cv2.inRange(img_hsv, min_hsv, max_hsv) 
    _, line = cv2.threshold(line, 248, 255, cv2.THRESH_BINARY)
````
![](https://github.com/bbeloqui/Robotica/blob/main/Follow_Line/vision_binaria.PNG)

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

Una vez que ya tenemos la obtención de la línea unicamente, el siguiente paso es obtener el contorno de la línea para poder "situarla" en la imagen. En mi caso, divido mi parte de interés en 2 trozos ("podemos decir que la parte entre las 2 líneas es lo que viene y desde la 2 línea hasta el final es lo que tenemos enfrente"). En la imagen que se muestra se puede ver lo antes explicado sobre las líneas.
Para la obtención de los bordes utilizamos la sentencia "cv2.findContours", posteriormente, si queremos podemos representarlo en la imagen que vamos devolviendo con la sentencia "cv2.drawContours"

````
    contorno_up, _ = cv2.findContours(c_up, cv2.RETR_LIST, cv2.CHAIN_APPROX_SIMPLE)
    contorno_down, _ = cv2.findContours(c_down, cv2.RETR_LIST, cv2.CHAIN_APPROX_SIMPLE)
    cv2.drawContours(img[y0:yf, x0:xf], contorno_up, -1, (0, 255, 0), 1)
    cv2.drawContours(img[y1: y1f, x1:x1f], contorno_down, -1, (0, 255, 0), 1)
````
Una vez que ya tenemos el contorno obtenido mediante la sentencia "cv2.moments(contorno)" obtenemos la coordenada X, Y del centroide. En mi caso, lo relizo 2 veces una para cada parte en la que tengo dividido mi parte de interés.
Ya tenemos las coordenadas de nuestros centroides por lo que podemos actualizarlas, ya que las habíamos inicializado a -1, si las coordenadas obtenidas son -1 se queda igual que las inicializadas ( en iteraciones siguientes, se quedará con el valor anterior obtenido) y si no..pues nos quedamos con las actuales (en cada iteración se actualizan las coodenadas de los centroides obtenidas).
Podemos obtener el error del centroide al centro, y así saber cuanto nos separamos de la vertical que sería el lugar perfecto del centroide (lo calculamos para ambos centroides) , X0 y X1 son 0 y mid_h es la mitad del tamaño de columnas de píxeles de la imagen.
````
    err = (x_u + x0) - mid_h
    err_b = (x_d + x1) - mid_h
````

![](https://github.com/bbeloqui/Robotica/blob/main/Follow_Line/vision_general.PNG)
                     
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------  
### PID 
Calcula un valor de error como la diferencia entre la salida deseada y la salida actual y aplica una correción basada en terminos proporcionales, integrales y derivados.
- kp = da una salida que es proporcional al error actual. Si el error es cero kp=0.
- ki = elimina el error de compensación que acumula el controlador kp. Integra el error durante un periodo de tiempo hasta que el valor del error llega a 0.
- kd = proporciona una salida que depende de la tasa de cambio o error con respecto al tiempo.

Traducido a código para la obtención del PID
````
    p_err = - kp * err
    d_err = - kd * (err - prev_err)
    i_err = - ki * accum_err
````
Donde err es el error del centroide superior, prev_err es el error de la iteración anterior y acumm_err el acumulado. Estos valores se irán actualizando en cada itación.
El PID obtenido será la suma de los 3 errores, que será nuestra W.
````
    pid = p_err + d_err + i_err
````
**Nota: fijando una vel. mínima y una vel. máxima podemos ir jugando con estos 3 parámetros para ver cual es el que más se ajusta al tomar la curva.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
            
Antes de calcular los ratios para modular la velocidad. Determinamos la distancia entre los centroides y la diferencia entre el error del centroide superior y el inferior.
 
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 
### Ratio de la velocidad angular
Para calcular el ángulo que se forma en el centroide superior con el centro de la imagen utilizamos la función ```` arctg(h/b)  ```` donde b es la distancia del centroide con el centro de la imagen y h es la distancia entre el centroide superior e inferior. Este valor se obtiene en radianes por lo que tendremos que pasarlo a grados. El valor resultante siempre será inferior a 90º , ya que si es 90º significa que los puntos están alineados. Ahora ya podemos obtener la razón que estará entre [0, 1]. Donde próximo a 0 será en curvas y próximo a 1 en las rectas.
````
    rads = math.atan(abs(h/(b + 1e-9))) # para que no sea cero la division
    alpha = math.degrees(rads)
    v_radio = alpha / 90
````
Calculamos el ratio tanto para el centroide superior (v_ratio0) como para el centroide inferior(v_ratio1).
````
    v_radio0 = velocidad_curva(h, b)
    v_radio1 = velocidad_curva(h, b1)
````

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

Una vez que tenemos ratio para la velocidad angular, introducimos una constante vc que nos permitirá regular la velocidad dentro de la curva. Esta constante mejora la estabilidad en las curvas, aunque se producen cambios bruscos de velocidad al terminar la curva.
````
    v_radio = vc * v_radio0 + (1 - vc) * v_radio1
````
Para suavizar estos cambios bruscos de velocidad se introduce la constante sv v_ratio_ant, en la primera iteración será un valor que determinamos nosotros y en las iteaciones siguientes será el valor procedente de la secuencia antes mencionada.
````
    v_radio = sv * v_radio + (1 - sv) * v_ratio_ant
````
Trás ésto ya podemos fijar la vel. min. y la vel. max. a la que irá el coche durante el circuito.
````
    v = max(v_min, v_max * v_radio)
````
Por último podemos dar la orden al coche de  que avance con las sentencias ````  HAL.setV(v) ```` y ````  HAL.setW(w) ```` 

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

### ¿No ve línea?
Para incluir robustez a nuestro coche, tenemos que tener encuenta el momento en el que el coche no detecta línea. La finalidad de la función creada es intentar encontrar el centroide superior y si no lo encuentra girar un poco y volver a ver si lo encuentra. Esto se realizará hasta que encuentre el centro y ya comenzará a ejecutarse el código de forma "normal".
````
vuelta = True

while vuelta:
    img_vuelta = HAL.getImage()
    GUI.showImage(img_vuelta)
    color_vuelta = filtro_color(img_vuelta)
    color_vuelta = color_vuelta[y0:yf, x0:xf]
    contorno_vuelta, _ = cv2.findContours(color_vuelta, cv2.RETR_LIST, cv2.CHAIN_APPROX_SIMPLE)
    for i in contorno_vuelta:
        M = cv2.moments(i)
        if M['m00'] != 0:
            x_v = int(M['m10'] / M['m00'])
            y_v = int(M['m01'] / M['m00'])
            
    if x_v == -1 or y_v == -1:
        HAL.setW(0.2)
    else:
        p_x_u = x_v
        p_y_u = x_v
        vuelta = False
````
### Vídeo de vuelta

Ahora se mostrará un vídeo del coche dando una vuelta. La velocidad es un valor que se pueden ir variando hasta encontrar una solución, la cual tenga el compromiso de ir lo más rápido posible, pero sin chocarse con los muros y siguiendo la línea lo más cerca posible.
Los valores con los que poder "jugar" son: kp, kd, ki, vc, sv, v_min, v_max.
Para el caso del vídeo, los valores son:
v_min = 9.5, v_max = 35, kp = 0.009, kd = 0.005, ki = 0.002, vc = 0.75, sv = 0.1.
Con estos parámetros el tiempo estimado es de 1.07 minutos.

![ ](https://github.com/bbeloqui/Robotica/blob/main/Follow_Line/video_4.mp4)

**Nota: es posible que no pueda verse el vídeo directamente por el peso del mismo, habrá que descargarlo.
