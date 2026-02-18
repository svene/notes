# Mockito Tips and Tricks

## Compare nested structures but ignore unimportant fields

````java
var taskCaptor = ArgumentCaptor.forClass(MyTask.class);
ArgumentCaptor<List<MultipartFile>> multiPartFilesCaptor = ArgumentCaptor.forClass(List.class);
verify(myApi).archive(
    eq("someName"),
    eq("307914242"),
    eq("true"),
    isNull(),
    archiveRequestCaptor.capture() // field 'task' of type MyRequest, contains nested field 'contents' of type List
);

assertThat(requestCaptor.getValue())
    .usingRecursiveComparison()
    .ignoringFields("field1", "contents.fileName", "task.myTaskProp1") // works for nested fields as well
    .isEqualTo(
        new MyTask()
            ...
            .contents(List.of(new MyContent(...)))
    );
````

