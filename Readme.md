﻿## LICENSE
See LICENSE.md file for more details

## INSTALATION

Clone project, add to your solution.
This setup is for server side Blazor, but for WASM blazor (client side) would be good as well.

Add line to your Blazor project `_Host.cshtml`

`<script>src="_content/PiNetwork.Blazor.Sdk/PiNetwork.Blazor.Sdk.js"</script>`
    
Add lines to your Blazor project `appsettings.json`

    "PiNetwork": {
        "ApiKey": "YourApiKey",
        "BaseUrl": "https://api.minepi.com/v2"
    }


Add reference to your Blazor project to this PiNetwork.Blazor.Sdk

Modify your Blazor project `Startup.cs` file:

    public void ConfigureServices(IServiceCollection services)
    {
    //...your rest code
    services.AddScoped<IPiNetworkClientBlazor, ServicesPiNetworkFacade>();
    services.AddScoped<IPiNetworkServerBlazor, PiNetworkServerBlazor>();
    //...your rest code

    services.AddPiNetwork(options =>
    {
            options.ApyKey = this.Configuration["PiNetwork:ApiKey"];
            options.BaseUrl = this.Configuration["PiNetwork:BaseUrl"];
    });
    
    //Add this two lines PiNetwork.Blazor.Sdk uses SessionStorage and MemoryCache
    services.AddBlazoredSessionStorage();
    services.AddMemoryCache();

    //...your rest code

    }
    
    // for Pi network browser to work fine, these lines should be added
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILoggerFactory loggerFactory)
    {

    //... your rest code

    app.UseRouting();
    app.UseCookiePolicy(
            new CookiePolicyOptions
            {
                MinimumSameSitePolicy = SameSiteMode.None,
                HttpOnly = HttpOnlyPolicy.Always,
                Secure = CookieSecurePolicy.Always
            }
    );
    app.UseAuthentication();
    app.UseAuthorization();
    
    //... your rest code
    }

Now you will need to create class to deal with PiNetwPiNetwork.Blazor.Sdk callbacks. This class must be extended from abstract class PiNetworkClientBlazor. You will need to override some methods to your needs. In example bellow your bussiness logic handler is IOrderServices.

Put this new class to you Blazor project. Lets call this new class ServicesPiNetworkFacade

    public class ServicesPiNetworkFacade : PiNetworkClientBlazor
    {
        private readonly ILogger logger;
        private readonly IPiNetworkServerBlazor server;
        private readonly IOrderServices orderServices;
        
        //If you don't need logging use constructor without ILoggerFactory
        public ServicesPiNetworkFacade(ILoggerFactory loggerFactory,
                                    NavigationManager navigationManager,
                                    IJSRuntime jsRuntime,
                                    ISessionStorageService sessionStorage,
                                    IPiNetworkServerBlazor server,
                                    IOrderServices orderServices)
                : base(navigationManager, jsRuntime, sessionStorage, loggerFactory)
        {
            this.logger = loggerFactory.CreateLogger(this.GetType().Name);
            this.server = server;
            this.orderServices = orderServices; //your bussiness logic is here.
        }

        public override async Task CreatePaymentOnReadyForServerApprovalCallBack(string paymentId)
        {
            //...you code
            var result = await this.server.PaymentApprove(paymentId); //don't change this row.
            //...app bussiness logic
            await this.orderServices.SavePaymentStatusPiNetwork(result.Metadata.OrderId, Enums.PaymentStatus.Waiting, null);
        }

        public override async Task CreatePaymentOnReadyForServerCompletionCallBack(string paymentId, string txid)
        {
            //..you code
            var result = await this.server.PaymentComplete(paymentId, txid); //don't change this row
            //...app bussiness logic
            await this.orderServices.SavePaymentStatusPiNetwork(result.Metadata.OrderId, Enums.PaymentStatus.AdditionalSuccess, txid);
            this.navigationManager.NavigateTo($"myorders/{result.Metadata.OrderId}", forceLoad: true);
        }

        public override Task AuthenticateOnErrorCallBack(object error)
        {
        }

        public override Task CreatePaymentOnCancelCallBack(string paymentId)
        {
        }

        public override Task CreatePaymentOnErrorCallBack(object error, PaymentDto payment)
        {
        }

        public override Task OpenShareDialog(string title, string message)
        {
        }
    }

## SEE REAL WORLD PRODUCTION EXAMPLE AND USE FOR YOUR APP
See ServicesPiNetworkFacadeExample.md

## RETRIES
By default if something fails PiNetworkClientBlazor will try to retry (if it is reasonable to do so and could solve problem) up to 10 times with 1000 milliseconds intervals.
You can override these values in ServicesPiNetworkFacade:

    public virtual int Retries { get; set; } = 10;

    public virtual int RetryDelay { get; set; } = 1000;

## USE ENUMS FOR MESSAGES IN PiNetworkConstantsEnums.cs
See PiNetwork.Blazor.Sdk.ConstantsEnums for messages to pass to front side. Messages stored in SessionStorage.

    AuthenticationError = 0,
    PaymentError = 1,
    AuthenticationSuccess = 2,
    PaymentSuccess = 3

## USE CONSTANTS IN PiNetworkConstantsEnums.cs
    public const string PiNetworkSdkCallBackError = "PiNetworkSdkCallBackError";
    public const string PiNetworkDoNotRedirect = "PiNetworkDoNotRedirect";
    public const string IsPiNetworkBrowser = "IsPiNetworkBrowser";

If you don't need to redirect use constant "PiNetworkDoNotRedirect" this is used in AuthenticateOnSuccessCallBack(AuthResultDto auth, string redirectUri).
See ServicesPiNetworkFacadeExample.md.

## OPEN SHARE DIALOG
Inject in your *.razor file '@inject IPiNetworkClientBlazor piClient'
    
    //OpenShareDialog(string title, string message)
    await this.piClient.OpenShareDialog("Share", "https://www.your-pi-network-project.com"))

## CHECK IF BROWSER IS PI NETWORK BROWSER
PiNetworkClientBlazor.IsPiNetworkBrowser() this method is not from PI SDK, but still very practical.

## HOW TO AUTHENTICATE AND MAKE PAYMENTS FROM YOUR BLAZOR APP?

### AUTHENTICATE
Inject in your *.razor file '@inject IPiNetworkClientBlazor piClient'

    await this.piClient.Authenticate(redirectUri, 0); // 0 - first attempt

If you don't need to redirect use ConstantsEnums.PiNetworkDoNotRedirect (if you provide just empty string it will redirect to "/").

### MAKE PAYMENTS
Inject in your *.razor file '@inject IPiNetworkClientBlazor piClient'

First authenticate with ConstantsEnums.PiNetworkDoNotRedirect option. Authenticate every time before making payment.

    await this.piClient.Authenticate(ConstantsEnums.PiNetworkDoNotRedirect, 0); // 0 - first attempt

Now make payment

    //memo - order discription. Don't exceed 25 characters.
    //public virtual async Task CreatePayment(decimal amount, string memo, int orderId, int retries = 0)
    
    await this.piClient.CreatePayment(total, memo, OrderId);

    

    



    




