[基础使用](https://www.jianshu.com/p/435ca449da41)

##### model默认后缀修改

![logo](0.png ':size=WIDTHxHEIGHT')


##### <font color="red"> 防止定义的泛型<T>解析被覆盖 </font>
执行Alt+J，这样就会自动清理掉整理json_convert_content.dart和api_response_entity.g.dart中的ApiResponseData痕迹。再把dynamic替换成T,并且去除顶部的@JsonSerializable()，避免下次执行Alt+J，替换掉自己的自定义。并且把api_response_entity.g.dart移除generated目录，因为那个目录会自动删除无用的文件。可以和api_reponse_entity.dart单独存放在一个文件夹当中。    

