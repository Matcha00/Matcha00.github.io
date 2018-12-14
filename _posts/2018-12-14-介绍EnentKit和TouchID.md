---
layout: post
titile: 入门EventKit和TouchID
date: 2018-12-14 15:00:24.000000000 +09:00
tags: 开发笔记
---

### EventKit

#### EventKit简介

EventKit是iOS事件管理库，通过EventKit可以向日历或提醒事项添加、删除、修改管理等功能

#### EventKit用法

	- (void)createEventCalendarTitle:(NSString *)title location:(NSString *)location startDate:(NSDate *)startDate endDate:(NSDate *)endDate allDay:(BOOL)allDay alarmArray:(NSArray *)alarmArray{
    __weak typeof(self) weakSelf = self;

    EKEventStore *eventStore = [[EKEventStore alloc] init];

    if ([eventStore respondsToSelector:@selector(requestAccessToEntityType:completion:)])
    {
        [eventStore requestAccessToEntityType:EKEntityTypeEvent completion:^(BOOL granted, NSError *error){

            dispatch_async(dispatch_get_main_queue(), ^{
                __strong typeof(weakSelf) strongSelf = weakSelf;
                if (error)
                {
                    [strongSelf showAlert:@"添加失败，请稍后重试"];

                }else if (!granted){
                    [strongSelf showAlert:@"不允许使用日历,请在设置中允许此App使用日历"];

                }else{
                    EKEvent *event  = [EKEvent eventWithEventStore:eventStore];
                    event.title     = title;
                    event.location = location;

                    NSDateFormatter *tempFormatter = [[NSDateFormatter alloc]init];
                    [tempFormatter setDateFormat:@"dd.MM.yyyy HH:mm"];

                    event.startDate = startDate;
                    event.endDate   = endDate;
                    event.allDay = allDay;

                    //添加提醒
                    if (alarmArray && alarmArray.count > 0) {

                        for (NSString *timeString in alarmArray) {
                            [event addAlarm:[EKAlarm alarmWithRelativeOffset:[timeString integerValue]]];
                        }
                    }

                    [event setCalendar:[eventStore defaultCalendarForNewEvents]];
                    NSError *err;
                    [eventStore saveEvent:event span:EKSpanThisEvent error:&err];
                    [strongSelf showAlert:@"已添加到系统日历中"];

                }
            });
        }];
    }
    
   
	}


EventKit所有关里需要实例化一个EKEventStore实例，(建议将EKEventStore设置为单利)，通过EKEventStore实例创建EKEvent(添加日历事件)或EKReminder(提醒事件)根据响应api去操作
	

   
  
### TouchID或FaceID

	+ (instancetype)sharedInstance
    {
    static WWTouchIDOfFaceID *wwID = nil;
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        wwID = [[WWTouchIDOfFaceID alloc]init];
    });
    
    return wwID;
    }

	- (void)showWWIDWithDescribe:(NSString *)describe block:(WWIDStateBlock)block
	{
	    if (!describe) {
	//        if(iPhoneX){
	//            describe = @"验证已有面容";
	//        }else{
	            describe = @"通过Home键验证已有指纹";
	        //}
	    }
	    
	    if (NSFoundationVersionNumber < NSFoundationVersionNumber_iOS_8_0) {
        
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"系统版本不支持TouchID/FaceID (必须高于iOS 8.0才能使用)");
            block(WWIDStateNotSupport, nil);
        });
        
        return;
    }
    
    LAContext *context = [[LAContext alloc] init];
    
    context.localizedFallbackTitle = @"输入密码";
    
    NSError *error = nil;
    
    if ([context canEvaluatePolicy:LAPolicyDeviceOwnerAuthentication error:&error]) {
        [context evaluatePolicy:LAPolicyDeviceOwnerAuthentication localizedReason:describe reply:^(BOOL success, NSError * _Nullable error) {
            if (success) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    NSLog(@"TouchID/FaceID 验证成功");
                    block(WWIDStateSuccess, error);
                });
            }else if (error) {
                if (@available(iOS 11.0, *)) {
                    switch (error.code) {
                        case LAErrorAuthenticationFailed:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"TouchID/FaceID 验证失败");
                                block(WWIDStateFail, error);
                            });
                            break;
                        }
                        case LAErrorUserCancel:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"TouchID/FaceID 被用户手动取消");
                                block(WWIDStateUserCancel, error);
                            });
                        }
                            break;
                        case LAErrorUserFallback:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"用户不使用TouchID/FaceID,选择手动输入密码");
                                block(WWIDStateInputPassword, error);
                            });
                        }
                            break;
                        case LAErrorSystemCancel:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"TouchID/FaceID 被系统取消 (如遇到来电,锁屏,按了Home键等)");
                                block(WWIDStateSystemCancel, error);
                            });
                        }
                            break;
                        case LAErrorPasscodeNotSet:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"TouchID/FaceID 无法启动,因为用户没有设置密码");
                                block(WWIDStatePasswordNotSet, error);
                            });
                        }
                            break;
                        case LAErrorBiometryNotEnrolled:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"TouchID/FaceID 无法启动,因为用户没有设置TouchID/FaceID");
                                block(WWIDStateTouchIDNotSet, error);
                            });
                        }
                            break;
                        case LAErrorBiometryNotAvailable:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"TouchID/FaceID 无效");
                                block(WWIDStateTouchIDNotAvailable, error);
                            });
                        }
                            break;
                        case LAErrorBiometryLockout:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"TouchID/FaceID 被锁定(连续多次验证TouchID/FaceID失败,系统需要用户手动输入密码)");
                                block(WWIDStateTouchIDLockout, error);
                            });
                        }
                            break;
                        case LAErrorAppCancel:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"当前软件被挂起并取消了授权 (如App进入了后台等)");
                                block(WWIDStateAppCancel, error);
                            });
                        }
                            break;
                        case LAErrorInvalidContext:{
                            dispatch_async(dispatch_get_main_queue(), ^{
                                NSLog(@"当前软件被挂起并取消了授权 (LAContext对象无效)");
                                block(WWIDStateInvalidContext, error);
                            });
                        }
                            break;
                        default:
                            break;
                    }
                }
            }
        }];
    } else {
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"当前设备不支持TouchID/FaceID");
            block(WWIDStateNotSupport, error);
        });
    }
	}
	@end
	
核心代码

    LAContext *context = [[LAContext alloc] init];
    // 本地验证框架上下文
    
    context.localizedFallbackTitle = @"输入密码";
    
    NSError *error = nil;
    
    if ([context canEvaluatePolicy:LAPolicyDeviceOwnerAuthentication error:&error]) {
    // 是否支持TouchID或FaceID
        [context evaluatePolicy:LAPolicyDeviceOwnerAuthentication localizedReason:describe reply:^(BOOL success, NSError * _Nullable error)
        //验证状态



