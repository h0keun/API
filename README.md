## API
ImageLoad &amp; movieApi

### ImageLoad
응답으로 받는 데이터가 이미지파일일 경우에는 이미지파일의 크기가 1mb가 넘는 경우가 많기 때문에 이미지파일을 다른 데이터와 구분하여 받는 방식이 더 효율적이다. 따라서 이미지 파일에대한 정보만 응답데이터에 넣어두고 이미지 파일은 별도로 다운로드한다.  

현업에서는 보통 Glide이나 Picaso같은 이미지 로딩 라이브러리를 사용한다. 본예제에서는 외부라이브러리 없이 직접 이미지를 다운로드하는 코드를 구성한다.  
1. AsyncTask로 이미지 다운
  스레드를 사용하기위해 AsyncTask를 상속하여 새로운 클래스를 정의하고 이 클래스를 이용해 객체를 생성할 때 [이미지가 있는 주소]와 함께 이 이미지를 다운로드받은 후 화면에 보여줄 때 사용할 [이미지뷰(ImageView) 객체]를 파라미터로 전달한다.
  ```JAVA
  public class ImageLoadTask extends AsyncTask<Void, Void, Bitmap> {
    private String urlStr;
    private ImageView imageView;

    public ImageLoadTask(String urlStr, ImageView imageView) {
        this.urlStr = urlStr;
        this.imageView = imageView;
    }
  ```
  스레드 내에서 동작하는 doInBackground 메소드 안에서는 웹서버의 이미지 데이터를 받아 비트맵 객체로 만들어준다. BitmapFactory 클래스의 decodeStream 메소드를 사용하면 간단한 코드 만으로도 비트맵 객체를 만들어줄 수 있다.
  ```JAVA
  @Override
  protected Bitmap doInBackground(Void... voids) {
      Bitmap bitmap = null;
     try {
         URL url = new URL(urlStr);
         bitmap = BitmapFactory.decodeStream
         (url.openConnection().getInputStream());
         }catch(Exception e) {
          e.printStackTrace();
   }
    return bitmap;
  }
  ```
  비트맵 객체로 변환하고 나면 메인 스레드에서 이미지뷰에 표시힌디. onPostExecute 메소드 안에 다음과 같이 넣어준다.
  ```JAVA
  @Override
  protected void onPostExecute(Bitmap bitmap) {
      super.onPostExecute(bitmap);

      imageView.setImageBitmap(bitmap);
      imageView.invalidate();
  }
  ```
2. 비트맵 객체의 메모리 해제
  비트맵 객체는 메모리에 만들어진 후 해제되지 않으면 메모리에 계속 남아있게 된다. 자바의 Garbage Collection 메커니즘을 이용해 메모리에서 해제될 수 있지만 앱에서 여러 이미지를 로딩하게 되면 메모리가 부족해지는 문제가 발생할 수 있으므로 사용하지 않는 비트맵 객체는 recycle 메소드를 이용해 즉시 해제시키는 것이 필요하다.  
  AsyncTask를 상속하여 만든 클래스 안에서 이미지를 다운로드하여 비트맵 객체로 만드는 경우 동일한 비트맵 객체를 요청한다면 다운로드한 이미지 파일을 로컬에 저장했다가 재사용할 수도 있고 아니면 이전 비트맵 객체를 메모리에서 해제한 후 새로 다운로드할 수도 있다.  
  본 예제에서는 이전 비트맵 객체를 메모리에서 해제한 후 새로 다운로드하는 방법으로 만든다. 클래스 안에 HashMap 객체를 만들고 이미지의 주소를 메모리에 만들어진 비트맵 객체와 매핑되도록 해 준다.
  ```JAVA
  public class ImageLoadTask extends AsyncTask<Void, Void, Bitmap> {
    private String urlStr;
    private ImageView imageView;

    private static HashMap<String, Bitmap> bitmapHash = new HashMap<String, Bitmap>();
  ```
  이미지 데이터를 이용해 비트맵 객체를 만들었을 때는 해시테이블에 그 객체를 추가한다. 그리고 새로운 비트맵 객체를 만들기 전에는 항상 해시테이블 안에 동일한 주소를 요청하는 경우에 이전에 만들어졌던 비트맵 객체를 메모리에서 해제하도록 한다.
  ```JAVA
  @Override
  protected Bitmap doInBackground(Void... voids) {
      Bitmap bitmap = null;
      try {
         if (bitmapHash.containsKey(urlStr)) {
              Bitmap oldBitmap = bitmapHash.remove(urlStr);
              if (oldBitmap != null) {
                  oldBitmap.recycle();
                  oldBitmap = null;
              }
          }

          URL url = new URL(urlStr);
          bitmap =  BitmapFactory.decodeStream(url.openConnection().getInputStream());

          bitmapHash.put(urlStr, bitmap);
      } catch(Exception e) {
          e.printStackTrace();
      }

      return bitmap;
  }
  ```
