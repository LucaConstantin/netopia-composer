<div align="center"><a href="https://netopia-payments.com/"><img alt="Parsedown" src="https://suport.mobilpay.ro/np-logo-blue.svg" width="240" /></a></div>

# NETOPIA Payments
## NETOPIA Payments Composer

### Installation
Run the following command from root of your project 
* <code>composer require netopia/payment</code>
#### or 
* add the **"netopia/payment"** to your **composer.json** file like the following example
    <code>

        "require": {
            ...
            "netopia/payment": "^1.1",
            ...
        }
    </code>
    
    and then run the following command from your Terminal
    
    <code>composer install</code>
    
### Example    

##### An example of Payment Request in Laravel 
<code>

    ...
    
    use Netopia\Payment\Address;
    use Netopia\Payment\Invoice;
    use Netopia\Payment\Request\Card;
    
    ...
    
    class ExampleController extends Controller
    {
        /**
         * all payment requests will be sent to the NETOPIA Payments server
         * SANDBOX : http://sandboxsecure.mobilpay.ro
         * LIVE : https://secure.mobilpay.ro
         */
        public $paymentUrl;
        /**
         * NETOPIA Payments is working only with Certificate. Each NETOPIA partner (merchant) has a certificate.
         * From your Admin panel you can download the Certificate.
         * is located in Admin -> Conturi de comerciant -> Detalii -> Setari securitate
         * the var $x509FilePath is path of your certificate in your platform
         * i.e: /home/certificates/public.cer
         */
        public $x509FilePath;
        /**
         * Billing Address
         */
        public $billingAddress;
        /**
         * Shipping Address
         */
        public $shippingAddress;
        
        ...
        
        public function index()
        {
            $this->paymentUrl   = 'http://sandboxsecure.mobilpay.ro';
            $this->x509FilePath = '/home/certificates/sandbox.XXXX-XXXX-XXXX-XXXX-XXXX.public.cer';
            try {
                $paymentRequest = new Card();
                $paymentRequest->signature  = 'XXXX-XXXX-XXXX-XXXX-XXXX';//signature - generated by mobilpay.ro for every merchant account
                $paymentRequest->orderId    = md5(uniqid(rand())); // order_id should be unique for a merchant account
                $paymentRequest->confirmUrl = 'https://example.test/card/success'; // is where mobilPay redirects the client once the payment process is finished and is MANDATORY
                $paymentRequest->returnUrl  = 'https://example.test/ipn';// is where mobilPay will send the payment result and is MANDATORY
    
                /*
                 * Invoices info
                 */
                $paymentRequest->invoice = new Invoice();
                $paymentRequest->invoice->currency  = 'RON';
                $paymentRequest->invoice->amount    = '20.00';
                $paymentRequest->invoice->tokenId   = null;
                $paymentRequest->invoice->details   = "Payment Via Composer library";
    
                /*
                 * Billing Info
                 */
                $this->billingAddress = new Address();
                $this->billingAddress->type         = "person"; //should be "person" / "company"
                $this->billingAddress->firstName    = "Billing name";
                $this->billingAddress->lastName     = "Billing LastName";
                $this->billingAddress->address      = "Bulevardul Ion Creangă, Nr 00";
                $this->billingAddress->email        = "test@billing.com";
                $this->billingAddress->mobilePhone  = "0732123456";
                $paymentRequest->invoice->setBillingAddress($this->billingAddress);
    
                /*
                 * Shipping
                 */
                $this->shippingAddress = new Address();
                $this->shippingAddress->type        = "person"; //should be "person" / "company"
                $this->shippingAddress->firstName   = "Shipping Name";
                $this->shippingAddress->lastName    = "Shipping LastName";
                $this->shippingAddress->address     = "Bulevardul Mihai Eminescu, Nr 00";
                $this->shippingAddress->email       = "test@shipping.com";
                $this->shippingAddress->mobilePhone = "0721234567";
                $paymentRequest->invoice->setShippingAddress($this->shippingAddress);
    
                /*
                 * encrypting
                 */
                $paymentRequest->encrypt($this->x509FilePath);
    
                /**
                 * send the following data to NETOPIA Payments server
                 * Method : POST
                 * Parameters : env_key, data, cipher, iv
                 * URL : $paymentUrl
                 */
                $env_key = $paymentRequest->getEnvKey();
                $data   = $paymentRequest->getEncData();
                $cipher = $paymentRequest->getCipher();
                $iv     = $paymentRequest->getIv();
            }catch (\Exception $e)
            {
                return "Oops, There is a problem!";
            }
        }
        ...
    }    
    
</code>

#### An example of IPN in Laravel

<code>
    
    ...
    
    use Netopia\Payment\Address;
    use Netopia\Payment\Invoice;
    use Netopia\Payment\Request\Card;
    use Netopia\Payment\Request\Notify;
    use Netopia\Payment\Request\PaymentAbstract;
    
    ...
    
    class IpnsController extends Controller 
    {
    
        ...
    
        public $errorCode;
        public $errorType;
        public $errorMessage;
        public $paymentUrl;
        public $x509FilePath;
        public $cipher;
        public $iv;
    
        ...
        
        public function index()
        {
        ...
        
            $this->errorType = PaymentAbstract::CONFIRM_ERROR_TYPE_NONE;
            $this->errorCode = 0;
            $this->errorMessage = '';
            $this->cipher     = 'rc4';
            $this->iv         = null;

            ....

            if(array_key_exists('cipher', $_POST))
            {
                $this->cipher = $_POST['cipher'];
                if(array_key_exists('iv', $_POST))
                {
                    $this->iv = $_POST['iv'];
                }
            }
    
            $this->paymentUrl = 'http://sandboxsecure.mobilpay.ro';
            $this->x509FilePath = '/home/certificates/sandbox.XXXX-XXXX-XXXX-XXXX-XXXXprivate.key';
    
    
            if (strcasecmp($_SERVER['REQUEST_METHOD'], 'post') == 0){
                if(isset($_POST['env_key']) && isset($_POST['data'])){
                    try {
                        $paymentRequestIpn = PaymentAbstract::factoryFromEncrypted($_POST['env_key'], $_POST['data'], $this->x509FilePath, null, $this->cipher, $this->iv);
                        $rrn = $paymentRequestIpn->objPmNotify->rrn;
                        if ($paymentRequestIpn->objPmNotify->errorCode == 0) {
                            switch($paymentRequestIpn->objPmNotify->action){
                                case 'confirmed':
                                    //update DB, SET status = "confirmed/captured"
                                    $this->errorMessage = $paymentRequestIpn->objPmNotify->errorMessage;
                                    break;
                                case 'confirmed_pending':
                                    //update DB, SET status = "pending"
                                    $this->errorMessage = $paymentRequestIpn->objPmNotify->errorMessage;
                                    break;
                                case 'paid_pending':
                                    //update DB, SET status = "pending"
                                    $this->errorMessage = $paymentRequestIpn->objPmNotify->errorMessage;
                                    break;
                                case 'paid':
                                    //update DB, SET status = "open/preauthorized"
                                    $this->errorMessage = $paymentRequestIpn->objPmNotify->errorMessage;
                                    break;
                                case 'canceled':
                                    //update DB, SET status = "canceled"
                                    $this->errorMessage = $paymentRequestIpn->objPmNotify->errorMessage;
                                    break;
                                case 'credit':
                                    //update DB, SET status = "refunded"
                                    $this->errorMessage = $paymentRequestIpn->objPmNotify->errorMessage;
                                    break;
                                default:
                                    $errorType = PaymentAbstract::CONFIRM_ERROR_TYPE_PERMANENT;
                                    $this->errorCode = PaymentAbstract::ERROR_CONFIRM_INVALID_ACTION;
                                    $this->errorMessage = 'mobilpay_refference_action paramaters is invalid';
                            }
                        }else{
                            //update DB, SET status = "rejected"
                            $this->errorMessage = $paymentRequestIpn->objPmNotify->errorMessage;
                        }
                    }catch (\Exception $e) {
                        $this->errorType = PaymentAbstract::CONFIRM_ERROR_TYPE_TEMPORARY;
                        $this->errorCode = $e->getCode();
                        $this->errorMessage = $e->getMessage();
                    }
    
                }else{
                    $this->errorType = PaymentAbstract::CONFIRM_ERROR_TYPE_PERMANENT;
                    $this->errorCode = PaymentAbstract::ERROR_CONFIRM_INVALID_POST_PARAMETERS;
                    $this->errorMessag = 'mobilpay.ro posted invalid parameters';
                }
    
            } else {
                $this->errorType = PaymentAbstract::CONFIRM_ERROR_TYPE_PERMANENT;
                $this->errorCode = PaymentAbstract::ERROR_CONFIRM_INVALID_POST_METHOD;
                $this->errorMessage = 'invalid request metod for payment confirmation';
            }
    
            /**
             * Communicate with NETOPIA Payments server
             */
    
            header('Content-type: application/xml');
            echo "<?xml version=\"1.0\" encoding=\"utf-8\"?>\n";
            if($this->errorCode == 0)
            {
                echo "<crc>{$this->errorMessage}</crc>";
            }
            else
            {
                echo "<crc error_type=\"{$this->errorType}\" error_code=\"{$this->errorCode}\">{$this->errorMessage}</crc>";
            }
    
        }
    }
</code>

##### Note / Suggestions
* if there is issue with namespace in your platform , you can solve it by getting help from Service Providers. 
for ex. in Laravel you can define a provider and put in your vendor and then set your namespace from the composer.json

* if in any case the Country , City , Zip code , ... is separated from the Address in your application , please merge it with Address and create full address for Billing/Shipping address.