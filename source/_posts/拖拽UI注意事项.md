# 拖拽UI

需要注意canvas处于什么模式下，一般是Scale with screen size

这时候需要考虑canvas的scale factor，父坐标，屏幕坐标之间的转换问题，也就是说拖拽时给的是屏幕坐标

需要把屏幕坐标转换为ui坐标

unity提供了一个API

```c#
public static bool ScreenPointToLocalPointInRectangle(RectTransform rect, Vector2 screenPoint, Camera cam, out Vector2 localPoint);
```

> Transform a screen space point to a position in the local space of a RectTransform that is on the plane of its rectangle.
>
> The cam parameter should be the camera associated with the screen point. For a RectTransform in a Canvas set to Screen Space - Overlay mode, the cam parameter should be null.
>
> When ScreenPointToLocalPointInRectangle is used from within an event handler that provides a PointerEventData object, the correct camera can be obtained by using PointerEventData.enterEventData (for hover functionality) or PointerEventData.pressEventCamera (for click functionality). This will automatically use the correct camera (or null) for the given event.

注意！事件系统提供的`pointerEventData.delta`是屏幕坐标下的位移，如果直接在OnDrag（）里操作anchoredPosition+=这个量得到的结果是错误的。

可以使用这个

offset是为了保证拖拽ui时ui不会自动跑到鼠标指针中心，保持鼠标和ui的相对位置。

```c#
Vector2 offset;

public void OnPointerDown(PointerEventData eventData)
{
    RectTransformUtility.ScreenPointToLocalPointInRectangle(
        (RectTransform)transform.parent,
        eventData.position, eventData.pressEventCamera,
        out var localMousePos);

    offset = (Vector2)transform.localPosition - localMousePos;
}

public void OnDrag(PointerEventData eventData)
{
    RectTransformUtility.ScreenPointToLocalPointInRectangle(
        (RectTransform)transform.parent,
        eventData.position, eventData.pressEventCamera,
        out var localMousePos);

    transform.localPosition = localMousePos + offset;
}
```

## 制作Drop效果

实现IDropHandler接口

使用pointerEventData.pointerDrag获得被拖拽的对象

```c#
public class ItemSlot : MonoBehaviour, IDropHandler
{
    public void OnDrop (PointerEventData eventData)
    {
        if (eventData.pointerDrag != null)
        {
            eventData.pointerDrag.GetComponent<RectTransform>().anchoredPosition = 
                GetComponent<RectTransform>().anchoredPosition
        }
    }
}
```