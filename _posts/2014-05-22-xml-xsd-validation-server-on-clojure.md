---
layout: post
title: Сервер валидации XML по XSD схеме на Clojure
comments: true
---
Для работы мне понадобилось написать xsd-схему для валидации xml-документа кастомера.

И все было бы не так страшно, потому как для проверки корректности собственной схемы - бери да пользуйся безумным
количеством различных онлайн-сервисов, но, как вы уже догадались, возникли проблемы.

Схема должна соответствовать стандарту [XSD v1.1](http://www.w3.org/TR/xmlschema11-1/)
(который от 2012 года).
Таких сервисов в интернете я не нашел, возможно, конечно, плохо искал, и тем не менее осталась реальная
задача - написать небольшое web-приложение, валидирующее xml по заданной xsd-схеме.

Так получилось, что сейчас я изучаю Clojure в контексте web development`а.

Само собой, лучшее изучения языка - практика и написание чего-нибудь рабочего.
Недолго думая, решил действовать.
Вылилось это в итоге в [репозиторий на гитхабе](https://github.com/chidori/xvs)

Решать задачу начал с поиска Clojure-библиотек, которые умеют валидировать xml по схеме версии 1.1, но таковых
не обнаружил, а то что в поиске нашлось было непонятно, как и с чем кушать, да и можно ли это вообще будет съесть.

В поиске Java-библиотек (из Clojure легко работать с Java) нашелся проект [Apache Xerces](http://xerces.apache.org/),
который представляет из себя целое семейство продуктов, в том числе и библиотека
[Apache Xerces2 Java](http://xerces.apache.org/xerces2-j/). Инструкция как сделать валидацию есть в
[FAQ](http://xerces.apache.org/xerces2-j/faq-xs.html#faq-1), мне очень пригодился кусок примера оттуда, в последствии
перенесённый в Clojure.

Подключил нужный мне последний `xerces/xercesImpl "2.11.0"` в `project.clj` и запустил в ожидании чуда...

Ответом мне был посыл в места не столь отдалённые в виде exception`а, кричащего о том, что SchemaFactory не в курсе о
схеме версии 1.1.

И снова пришлось обратиться к сайту Xerces2, где нашел информацию о том, что поддержка версии схемы 1.1 есть, но она  - beta, а beta
в maven`е не лежит (Clojure свои зависимости ищет в clojars - своём репозитории для clojure-библиотек и потом в maven
для java-библиотек, потому как построение имени происходит maven-like: groupId/artifactId).

Тогда появилась новая задача: подключить конкретную java-библиотеку к проекту.
Первая мысль, которая пришла в голову - classpath. Первым вариантом была библиотека
[pomegranate](https://github.com/cemerick/pomegranate), но коллега подсказал, что лучше использовать плагин для
[leiningen](http://leiningen.org/) - [lein-localrepo](https://github.com/kumarshantanu/lein-localrepo),
который идеально решил мою проблему.

Этот плагин добавляет .jar файл в локальный maven-репозиторий под указанной тобою версией.

После выполнения процедур добавления я смог успешно запустить проект, и он не сругался мне на схему версии 1.1, зато
выдал ошибку на отсутствие другой библиотеки. Она также была добавлена через плагин lein-localrepo.

Обе подключаемых библиотеки можно найти в папке [`lib/`](https://github.com/chidori/xvs/tree/master/lib) проекта.
Без них сервер работать не будет, их обязательно нужно добавить себе в локальный maven-репозиторий.

С клиентской стороны были сделаны 2 textarea, кнопка валидации и div`очка с output сообщением - большего не
потребовалось.

Все самое интересное происходит буквально в двух методах:

{% highlight clojure %}
(def schema-v11 "http://www.w3.org/XML/XMLSchema/v1.1")

(defn get-schema-validator [schema]
  "Create schema validator"
  (try
    (let [schema (if (nil? schema) (str "") schema)
          validator (-> (SchemaFactory/newInstance schema-v11)
                        (.newSchema (StreamSource. (StringReader. schema)))
                        (.newValidator))]
      {:status "ok" :message nil :validator validator})
    (catch SAXException e
      {:status "schema-error" :message (str (.getMessage e)) :validator nil})))
{% endhighlight %}
в этом методе идет попытка создания валидатора по переданной схеме.

Если схему не получилось создать и произошел Exception - мы сохраним его сообщение, и это будет проверкой схемы на её
корректность.

{% highlight clojure %}
(defn validate [xml schema request]
  (let [{:keys [status message validator]} (get-schema-validator schema)]
    (if (nil? validator)
      (json/write-str {:status status :message message})
      (try
        (.validate validator
            (StreamSource. (StringReader. (if (nil? xml) (str "") xml))))
        (json/write-str {:status "ok" :message nil})
        (catch SAXException e
          (json/write-str {:status "xml-error" :message (.getMessage e)}))))))
{% endhighlight %}
Этот метод - handler для post запроса на валидацию.

Если валидатор был создан нормально, то мы пытаемся валидировать XML, и если возник Exception - значит XML не прошел валидацию и мы вернем это сообщение.

Если же в обеих случаях не было никаких Exception - это значит, что схема прошла валидацию и xml ей всецело соответствует.

Если вы сталкивались с такой же задачей, и у вас есть другое решение, напишите. Мне будет очень интересно.