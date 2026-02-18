# AssertJ Tips and Tricks

## exception assertions

````java
assertThatExceptionOfType(IllegalArgumentException.class)
  .isThrownBy(() -> minioConnector.assertThatBucketExists(minioClient, "non-existing-bucket"))
  .withFailMessage("Could not connect to MinIO")
  .havingRootCause()
  .withMessage("The bucket does not exist in MinIO. Bucket name: " + "non-existing-bucket")
;
````

## java.util.Optional

````java
assertThat(maybeDto)
  .isPresent()
  .hasValueSatisfying(it -> assertThat(it.myId()).isEqualTo("009170010010200200100"))
  ;
````

## Lists

````java
List<Long> expected = Arrays.asList(4L);
List<Long> actual = ...;
assertThat(actual).containsExactlyElementsOf(expected);
````



