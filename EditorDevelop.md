# 1-MessageDialog OR Notification

```c++
    #include "Misc/MessageDialog.h"
    #include "Widgets/Notifications/SNotificationList.h"
    #include "Framework/Notifications/NotificationManager.h"

        EAppReturnType::Type ShowMsgDialog(EAppMsgType::Type MsgType, const FString& Msg, bool ShowAsWarning = true)
    {
        if(ShowAsWarning)
        {
            FText MsgTitle = FText::FromString("Tile desu");
            return FMessageDialog::Open(MsgType, FText::FromString(Msg), &MsgTitle);
        }
        else
        {
            return FMessageDialog::Open(MsgType, FText::FromString(Msg));
        }
        
    }

    void ShowNotifyInfo(const FString& Msg)
    {
        FNotificationInfo NotifyInfo(FText::FromString(Msg));
        NotifyInfo.FadeOutDuration = 7.f;
        NotifyInfo.bUseLargeFont = true;

        FSlateNotificationManager::Get().AddNotification(NotifyInfo);
    }
```
