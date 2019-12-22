备份一下博客里面用到的文章，万一那天丢失了还能恢复一下。顺便提供一个图床用。

下面这是按照我的编辑习惯，替换markdown里面用相对路径引用的文章，导出后直接复制到博客的编辑器里
```java
public static void main(String[] args) throws IOException {
        // 项目的根路径
        String title = "https://github.com/superzhangzl/Essential-Image-Optimization-translation/tree/master/";
        // 输入文件
        String inputFile = "...";
        // 输出文件路径
        String outputFile = "...";
        Stream<String> lines = Files.lines(Paths.get(inputFile));
        String pages = lines
                .map(line -> {
                    if (line.matches("!\\[.*\\]\\(.*\\)")) {
                        System.out.println(line);
                        return line.replace("(", "(" + title)
                                .replace(")", "?raw=true)")
                                .replace("/tree/", "/blob/");
                    } else {
                        return line;
                    }
                })
                .collect(Collectors.joining("\n"));
        // util 工具，要是没导入jar包重新写一个就是了
        FileUtils.write(new File(outputFile), pages, "UTF-8");
    }
```