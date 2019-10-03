<!-- markdownlint-disable MD002 MD041 -->

<span data-ttu-id="da1ef-101">この演習では、Azure AD での認証をサポートするために、前の手順で作成したアプリケーションを拡張します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-101">In this exercise you will extend the application from the previous exercise to support authentication with Azure AD.</span></span> <span data-ttu-id="da1ef-102">これは、Microsoft Graph を呼び出すために必要な OAuth アクセストークンを取得するために必要です。</span><span class="sxs-lookup"><span data-stu-id="da1ef-102">This is required to obtain the necessary OAuth access token to call the Microsoft Graph.</span></span> <span data-ttu-id="da1ef-103">これを行うには、 [iOS 用 Microsoft Authentication Library (MSAL)](https://github.com/AzureAD/microsoft-authentication-library-for-objc)をアプリケーションに統合します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-103">To do this, you will integrate the [Microsoft Authentication Library (MSAL) for iOS](https://github.com/AzureAD/microsoft-authentication-library-for-objc) into the application.</span></span>

1. <span data-ttu-id="da1ef-104">**Authsettings. plist**という名前の**graphtutorial**プロジェクトに、新しい**プロパティリスト**ファイルを作成します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-104">Create a new **Property List** file in the **GraphTutorial** project named **AuthSettings.plist**.</span></span>
1. <span data-ttu-id="da1ef-105">次の項目を**ルート**ディクショナリのファイルに追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-105">Add the following items to the file in the **Root** dictionary.</span></span>

    | <span data-ttu-id="da1ef-106">Key</span><span class="sxs-lookup"><span data-stu-id="da1ef-106">Key</span></span> | <span data-ttu-id="da1ef-107">型</span><span class="sxs-lookup"><span data-stu-id="da1ef-107">Type</span></span> | <span data-ttu-id="da1ef-108">値</span><span class="sxs-lookup"><span data-stu-id="da1ef-108">Value</span></span> |
    |-----|------|-------|
    | `AppId` | <span data-ttu-id="da1ef-109">String</span><span class="sxs-lookup"><span data-stu-id="da1ef-109">String</span></span> | <span data-ttu-id="da1ef-110">Azure portal からのアプリケーション ID</span><span class="sxs-lookup"><span data-stu-id="da1ef-110">The application ID from the Azure portal</span></span> |
    | `GraphScopes` | <span data-ttu-id="da1ef-111">配列</span><span class="sxs-lookup"><span data-stu-id="da1ef-111">Array</span></span> | <span data-ttu-id="da1ef-112">2つの文字列`User.Read`値:`Calendars.Read`</span><span class="sxs-lookup"><span data-stu-id="da1ef-112">Two String values: `User.Read` and `Calendars.Read`</span></span> |

    ![Xcode の AuthSettings plist ファイルのスクリーンショット](./images/auth-settings.png)

> [!IMPORTANT]
> <span data-ttu-id="da1ef-114">Git などのソース管理を使用している場合は、アプリ ID が誤ってリークしないように、ソース管理から**Authsettings の設定**ファイルを除外することをお勧めします。</span><span class="sxs-lookup"><span data-stu-id="da1ef-114">If you're using source control such as git, now would be a good time to exclude the **AuthSettings.plist** file from source control to avoid inadvertently leaking your app ID.</span></span>

## <a name="implement-sign-in"></a><span data-ttu-id="da1ef-115">サインインの実装</span><span class="sxs-lookup"><span data-stu-id="da1ef-115">Implement sign-in</span></span>

<span data-ttu-id="da1ef-116">このセクションでは、MSAL のプロジェクトを構成し、認証マネージャークラスを作成し、サインインしてサインアウトするためにアプリを更新します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-116">In this section you will configure the project for MSAL, create an authentication manager class, and update the app to sign in and sign out.</span></span>

### <a name="configure-project-for-msal"></a><span data-ttu-id="da1ef-117">MSAL のプロジェクトを構成する</span><span class="sxs-lookup"><span data-stu-id="da1ef-117">Configure project for MSAL</span></span>

1. <span data-ttu-id="da1ef-118">[コントロール] をクリックし、[**名前を付けて開く**]、[**ソースコード**] の順に選択し**ます。**</span><span class="sxs-lookup"><span data-stu-id="da1ef-118">Control click **Info.plist** and select **Open As**, then **Source Code**.</span></span>
1. <span data-ttu-id="da1ef-119">`<dict>`要素内に次のように追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-119">Add the following inside the `<dict>` element.</span></span>

    ```xml
    <key>CFBundleURLTypes</key>
    <array>
      <dict>
        <key>CFBundleURLSchemes</key>
        <array>
          <string>msauth.$(PRODUCT_BUNDLE_IDENTIFIER)</string>
        </array>
      </dict>
    </array>
    <key>LSApplicationQueriesSchemes</key>
    <array>
        <string>msauthv2</string>
        <string>msauthv3</string>
    </array>
    ```

1. <span data-ttu-id="da1ef-120">**Appdelegate**を開き、ファイルの先頭に次の import ステートメントを追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-120">Open **AppDelegate.m** and add the following import statement at the top of the file.</span></span>

    ```objc
    #import <MSAL/MSAL.h>
    ```

1. <span data-ttu-id="da1ef-121">次の関数を `AppDelegate` クラスに追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-121">Add the following function to the `AppDelegate` class.</span></span>

    ```objc
    - (BOOL)application:(UIApplication *)app
                openURL:(NSURL *)url
                options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options
    {
        return [MSALPublicClientApplication handleMSALResponse:url
                                             sourceApplication:options[UIApplicationOpenURLOptionsSourceApplicationKey]];
    }
    ```

### <a name="create-authentication-manager"></a><span data-ttu-id="da1ef-122">認証マネージャーを作成する</span><span class="sxs-lookup"><span data-stu-id="da1ef-122">Create authentication manager</span></span>

1. <span data-ttu-id="da1ef-123">「 **Authenticationmanager**」という名前の**graphtutorial**プロジェクトに新しい**cocoa タッチクラス**を作成します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-123">Create a new **Cocoa Touch Class** in the **GraphTutorial** project named **AuthenticationManager**.</span></span> <span data-ttu-id="da1ef-124">フィールド**のサブクラス**で [ **NSObject** ] を選択します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-124">Choose **NSObject** in the **Subclass of** field.</span></span>
1. <span data-ttu-id="da1ef-125">**Authenticationmanager .h**を開き、その内容を次のコードで置き換えます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-125">Open **AuthenticationManager.h** and replace its contents with the following code.</span></span>

    ```objc
    #import <Foundation/Foundation.h>
    #import <MSAL/MSAL.h>

    NS_ASSUME_NONNULL_BEGIN

    typedef void (^GetTokenCompletionBlock)(NSString* _Nullable accessToken, NSError* _Nullable error);

    @interface AuthenticationManager : NSObject

    + (id) instance;
    - (void) getTokenInteractivelyWithCompletionBlock: (GetTokenCompletionBlock)completionBlock;
    - (void) getTokenSilentlyWithCompletionBlock: (GetTokenCompletionBlock)completionBlock;
    - (void) signOut;

    @end

    NS_ASSUME_NONNULL_END
    ```

1. <span data-ttu-id="da1ef-126">**Authenticationmanager. m**を開き、その内容を次のコードで置き換えます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-126">Open **AuthenticationManager.m** and replace its contents with the following code.</span></span>

    ```objc
    #import "AuthenticationManager.h"

    @interface AuthenticationManager()

    @property NSString* appId;
    @property NSArray<NSString*>* graphScopes;
    @property MSALPublicClientApplication* publicClient;

    @end

    @implementation AuthenticationManager

    + (id) instance {
        static AuthenticationManager *singleInstance = nil;
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^ {
            singleInstance = [[self alloc] init];
        });

        return singleInstance;
    }

    - (id) init {
        if (self = [super init]) {
            // Get app ID and scopes from AuthSettings.plist
            NSString* authConfigPath =
            [NSBundle.mainBundle pathForResource:@"AuthSettings" ofType:@"plist"];
            NSString* bundleId = NSBundle.mainBundle.bundleIdentifier;
            NSDictionary* authConfig = [NSDictionary dictionaryWithContentsOfFile:authConfigPath];

            self.appId = authConfig[@"AppId"];
            self.graphScopes = authConfig[@"GraphScopes"];

            // Create the MSAL client
            self.publicClient = [[MSALPublicClientApplication alloc] initWithClientId:self.appId
                                                                        keychainGroup:bundleId
                                                                                error:nil];
        }

        return self;
    }

    - (void) getTokenInteractivelyWithCompletionBlock:(GetTokenCompletionBlock)completionBlock {
        // Call acquireToken to open a browser so the user can sign in
        [self.publicClient
         acquireTokenForScopes:self.graphScopes
         completionBlock:^(MSALResult * _Nullable result, NSError * _Nullable error) {

            // Check error
            if (error) {
                completionBlock(nil, error);
                return;
            }

            // Check result
            if (!result) {
                NSMutableDictionary* details = [NSMutableDictionary dictionary];
                [details setValue:@"No result was returned" forKey:NSDebugDescriptionErrorKey];
                completionBlock(nil, [NSError errorWithDomain:@"AuthenticationManager" code:0 userInfo:details]);
                return;
            }

            NSLog(@"Got token interactively: %@", result.accessToken);
            completionBlock(result.accessToken, nil);
        }];
    }

    - (void) getTokenSilentlyWithCompletionBlock:(GetTokenCompletionBlock)completionBlock {
        // Check if there is an account in the cache
        NSError* msalError;
        MSALAccount* account = [self.publicClient allAccounts:&msalError].firstObject;

        if (msalError || !account) {
            NSMutableDictionary* details = [NSMutableDictionary dictionary];
            [details setValue:@"Could not retrieve account from cache" forKey:NSDebugDescriptionErrorKey];
            completionBlock(nil, [NSError errorWithDomain:@"AuthenticationManager" code:0 userInfo:details]);
            return;
        }

        // Attempt to get token silently
        [self.publicClient
         acquireTokenSilentForScopes:self.graphScopes account:account
         completionBlock:^(MSALResult * _Nullable result, NSError * _Nullable error) {
             // Check error
             if (error) {
                 completionBlock(nil, error);
                 return;
             }

             // Check result
             if (!result) {
                 NSMutableDictionary* details = [NSMutableDictionary dictionary];
                 [details setValue:@"No result was returned" forKey:NSDebugDescriptionErrorKey];
                 completionBlock(nil, [NSError errorWithDomain:@"AuthenticationManager" code:0 userInfo:details]);
                 return;
             }

             NSLog(@"Got token silently: %@", result.accessToken);
             completionBlock(result.accessToken, nil);
         }];
    }

    - (void) signOut {
        NSError* msalError;
        NSArray* accounts = [self.publicClient allAccounts:&msalError];

        if (msalError) {
            NSLog(@"Error getting accounts from cache: %@", msalError.debugDescription);
            return;
        }

        for (id account in accounts) {
            [self.publicClient removeAccount:account error:nil];
        }
    }

    @end
    ```

### <a name="add-sign-in-and-sign-out"></a><span data-ttu-id="da1ef-127">サインインとサインアウトを追加する</span><span class="sxs-lookup"><span data-stu-id="da1ef-127">Add sign-in and sign-out</span></span>

1. <span data-ttu-id="da1ef-128">**SignInViewController**ファイルを開き、その内容を次のコードに置き換えます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-128">Open the **SignInViewController.m** file and replace its contents with the following code.</span></span>

    ```objc
    #import "SignInViewController.h"
    #import "SpinnerViewController.h"
    #import "AuthenticationManager.h"

    @interface SignInViewController ()
    @property SpinnerViewController* spinner;
    @end

    @implementation SignInViewController

    - (void)viewDidLoad {
        [super viewDidLoad];
        // Do any additional setup after loading the view.

        self.spinner = [SpinnerViewController alloc];
        [self.spinner startWithContainer:self];

        [AuthenticationManager.instance
         getTokenSilentlyWithCompletionBlock:^(NSString * _Nullable accessToken, NSError * _Nullable error) {
             dispatch_async(dispatch_get_main_queue(), ^{
                 [self.spinner stop];

                 if (error || !accessToken) {
                     // If there is no token or if there's an error,
                     // no user is signed in, so stay here
                     return;
                 }

                 // Since we got a token, user is signed in
                 // Go to welcome page
                 [self performSegueWithIdentifier: @"userSignedIn" sender: nil];
             });
        }];
    }

    - (IBAction)signIn {
        self.spinner = [SpinnerViewController alloc];
        [self.spinner startWithContainer:self];

        [AuthenticationManager.instance
         getTokenInteractivelyWithCompletionBlock:^(NSString * _Nullable accessToken, NSError * _Nullable error) {
             dispatch_async(dispatch_get_main_queue(), ^{
                 [self.spinner stop];

                 if (error || !accessToken) {
                     // Show the error and stay on the sign-in page
                     UIAlertController* alert = [UIAlertController
                                                 alertControllerWithTitle:@"Error signing in"
                                                 message:error.debugDescription
                                                 preferredStyle:UIAlertControllerStyleAlert];

                     UIAlertAction* okButton = [UIAlertAction
                                                actionWithTitle:@"OK"
                                                style:UIAlertActionStyleDefault
                                                handler:nil];

                     [alert addAction:okButton];
                     [self presentViewController:alert animated:true completion:nil];
                     return;
                 }

                 // Since we got a token, user is signed in
                 // Go to welcome page
                 [self performSegueWithIdentifier: @"userSignedIn" sender: nil];
             });
         }];
    }
    @end
    ```

1. <span data-ttu-id="da1ef-129">**WelcomeViewController**を開き、既存`signOut`の関数を次のように置き換えます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-129">Open **WelcomeViewController.m** and replace the existing `signOut` function with the following.</span></span>

    ```objc
    - (IBAction)signOut {
        [AuthenticationManager.instance signOut];
        [self performSegueWithIdentifier: @"userSignedOut" sender: nil];
    }
    ```

1. <span data-ttu-id="da1ef-130">変更を保存し、シミュレータでアプリケーションを再起動します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-130">Save your changes and restart the application in Simulator.</span></span>

<span data-ttu-id="da1ef-131">アプリにサインインすると、Xcode の [出力] ウィンドウにアクセストークンが表示されます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-131">If you sign in to the app, you should see an access token displayed in the output window in Xcode.</span></span>

![Xcode でアクセストークンが表示されている出力ウィンドウのスクリーンショット](./images/access-token-output.png)

## <a name="get-user-details"></a><span data-ttu-id="da1ef-133">ユーザーの詳細を取得する</span><span class="sxs-lookup"><span data-stu-id="da1ef-133">Get user details</span></span>

<span data-ttu-id="da1ef-134">このセクションでは、Microsoft Graph へのすべての呼び出しを保持するヘルパークラスを作成し、 `WelcomeViewController`を更新して、この新しいクラスを使用してログインしているユーザーを取得します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-134">In this section you will create a helper class to hold all of the calls to Microsoft Graph and update the `WelcomeViewController` to use this new class to get the logged-in user.</span></span>

1. <span data-ttu-id="da1ef-135">**Authenticationmanager .h**を開き、次`#import`のステートメントをファイルの先頭に追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-135">Open **AuthenticationManager.h** and add the following `#import` statement at the top of the file.</span></span>

    ```objc
    #import <MSGraphMSALAuthProvider/MSGraphMSALAuthProvider.h>
    ```

1. <span data-ttu-id="da1ef-136">宣言内に`@interface`次の行を追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-136">Add the following line inside the `@interface` declaration.</span></span>

    ```objc
    - (MSALAuthenticationProvider*) getGraphAuthProvider;
    ```

1. <span data-ttu-id="da1ef-137">**Authenticationmanager. m**を開き、次の関数を`AuthenticationManager`クラスに追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-137">Open **AuthenticationManager.m** and add the following function to the `AuthenticationManager` class.</span></span>

    ```objc
    - (MSALAuthenticationProvider*) getGraphAuthProvider {
        // Create an MSAL auth provider for use with the Graph client
        return [[MSALAuthenticationProvider alloc]
                initWithPublicClientApplication:self.publicClient
                andScopes:self.graphScopes];
    }
    ```

1. <span data-ttu-id="da1ef-138">**Graphtutorial**という名前の**graphtutorial**プロジェクトに新しい**cocoa タッチクラス**を作成します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-138">Create a new **Cocoa Touch Class** in the **GraphTutorial** project named **GraphManager**.</span></span> <span data-ttu-id="da1ef-139">フィールド**のサブクラス**で [ **NSObject** ] を選択します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-139">Choose **NSObject** in the **Subclass of** field.</span></span>
1. <span data-ttu-id="da1ef-140">**Graphmanager**を開き、その内容を次のコードで置き換えます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-140">Open **GraphManager.h** and replace its contents with the following code.</span></span>

    ```objc
    #import <Foundation/Foundation.h>
    #import <MSGraphClientSDK/MSGraphClientSDK.h>
    #import <MSGraphClientModels/MSGraphClientModels.h>
    #import "AuthenticationManager.h"

    NS_ASSUME_NONNULL_BEGIN

    typedef void (^GetMeCompletionBlock)(MSGraphUser* _Nullable user, NSError* _Nullable error);

    @interface GraphManager : NSObject

    + (id) instance;
    - (void) getMeWithCompletionBlock: (GetMeCompletionBlock)completionBlock;

    @end

    NS_ASSUME_NONNULL_END
    ```

1. <span data-ttu-id="da1ef-141">**Graphmanager. m**を開き、その内容を次のコードで置き換えます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-141">Open **GraphManager.m** and replace its contents with the following code.</span></span>

    ```objc
    #import "GraphManager.h"

    @interface GraphManager()

    @property MSHTTPClient* graphClient;

    @end

    @implementation GraphManager

    + (id) instance {
        static GraphManager *singleInstance = nil;
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^ {
            singleInstance = [[self alloc] init];
        });

        return singleInstance;
    }

    - (id) init {
        if (self = [super init]) {

            MSALAuthenticationProvider* authProvider =
            [AuthenticationManager.instance getGraphAuthProvider];

            // Create the Graph client
            self.graphClient = [MSClientFactory
                                createHTTPClientWithAuthenticationProvider:authProvider];
        }

        return self;
    }

    - (void) getMeWithCompletionBlock:(GetMeCompletionBlock)completionBlock {
        // GET /me
        NSString* meUrlString = [NSString stringWithFormat:@"%@/me", MSGraphBaseURL];
        NSURL* meUrl = [[NSURL alloc] initWithString:meUrlString];
        NSMutableURLRequest* meRequest = [[NSMutableURLRequest alloc] initWithURL:meUrl];

        MSURLSessionDataTask* meDataTask =
        [[MSURLSessionDataTask alloc]
            initWithRequest:meRequest
            client:self.graphClient
            completion:^(NSData *data, NSURLResponse *response, NSError *error) {
                if (error) {
                    completionBlock(nil, error);
                    return;
                }

                // Deserialize the response as a user
                NSError* graphError;
                MSGraphUser* user = [[MSGraphUser alloc] initWithData:data error:&graphError];

                if (graphError) {
                    completionBlock(nil, graphError);
                } else {
                    completionBlock(user, nil);
                }
            }];

        // Execute the request
        [meDataTask execute];
    }

    @end
    ```

1. <span data-ttu-id="da1ef-142">**WelcomeViewController**を開き、次`#import`のステートメントをファイルの先頭に追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-142">Open **WelcomeViewController.m** and add the following `#import` statements at the top of the file.</span></span>

    ```objc
    #import "SpinnerViewController.h"
    #import "GraphManager.h"
    #import <MSGraphClientModels/MSGraphClientModels.h>
    ```

1. <span data-ttu-id="da1ef-143">次のプロパティを`WelcomeViewController`インターフェイス宣言に追加します。</span><span class="sxs-lookup"><span data-stu-id="da1ef-143">Add the following property to the `WelcomeViewController` interface declaration.</span></span>

    ```objc
    @property SpinnerViewController* spinner;
    ```

1. <span data-ttu-id="da1ef-144">既存`viewDidLoad`のを次のコードに置き換えます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-144">Replace the existing `viewDidLoad` with the following code.</span></span>

    ```objc
    - (void)viewDidLoad {
        [super viewDidLoad];
        // Do any additional setup after loading the view.

        self.spinner = [SpinnerViewController alloc];
        [self.spinner startWithContainer:self];

        self.userProfilePhoto.image = [UIImage imageNamed:@"DefaultUserPhoto"];

        [GraphManager.instance
         getMeWithCompletionBlock:^(MSGraphUser * _Nullable user, NSError * _Nullable error) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [self.spinner stop];

                if (error) {
                    // Show the error
                    UIAlertController* alert = [UIAlertController
                                                alertControllerWithTitle:@"Error getting user profile"
                                                message:error.debugDescription
                                                preferredStyle:UIAlertControllerStyleAlert];

                    UIAlertAction* okButton = [UIAlertAction
                                               actionWithTitle:@"OK"
                                               style:UIAlertActionStyleDefault
                                               handler:nil];

                    [alert addAction:okButton];
                    [self presentViewController:alert animated:true completion:nil];
                    return;
                }

                // Set display name
                self.userDisplayName.text = user.displayName ? : @"Mysterious Stranger";
                [self.userDisplayName sizeToFit];

                // AAD users have email in the mail attribute
                // Personal accounts have email in the userPrincipalName attribute
                self.userEmail.text = user.mail ? : user.userPrincipalName;
                [self.userEmail sizeToFit];
            });
         }];
    }
    ```

<span data-ttu-id="da1ef-145">変更を保存して今すぐアプリを再起動すると、サインイン後にユーザーの表示名と電子メールアドレスで UI が更新されます。</span><span class="sxs-lookup"><span data-stu-id="da1ef-145">If you save your changes and restart the app now, after sign-in the UI is updated with the user's display name and email address.</span></span>
