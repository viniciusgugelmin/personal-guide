**(Portuguese)**

**O que é?**

CAPTCHA é um mecanismo de segurança projetado para proteger aplicações web contra abusos automatizados (GOOGLE, 2026). Na prática, ele ajuda a garantir que uma requisição está sendo realizada por um usuário real, prevenindo cenários como:

- tentativas de login por força bruta
- spam em formulários (ex: “Fale Conosco”)
- solicitações abusivas de redefinição de senha
- criação ou enumeração automatizada de contas

O Google reCAPTCHA é uma das soluções de CAPTCHA mais comuns e pode funcionar tanto por meio de:

- v2: desafio interativo (“Não sou um robô”)
- v3: pontuação de risco (invisível para o usuário)

**Como começar**

Para implementar o CAPTCHA, é necessário primeiro gerar um par de chaves pública/privada, criado através da administração do Google reCAPTCHA: https://developers.google.com/recaptcha. Ao criar as chaves, será necessário configurar:

- Label: um nome amigável para identificar onde o CAPTCHA está sendo utilizado
- Tipo:
    - v2 → baseado em desafio
    - v3 → baseado em pontuação
- Domínios: é possível restringir quais domínios podem usar as chaves
- Nome do projeto no Google Developers Console

*É recomendado ter uma chave separada para desenvolvimento local, sem restrição de domínio.*

Após criar o par de chaves, o Google fornece um painel de administração onde é possível monitorar:

- volume de uso
- validações bem-sucedidas vs falhas
- padrões de tráfego suspeitos e possíveis ataques

*O limite de uso gratuito é de 10.000 validações por mês. Acima desse limite, é necessário um plano pago.*

**Configuração no SAP Commerce**

Uma vez que você tenha as chaves, é necessário decidir como irá gerenciá-las:

- uma chave por site (recomendado para governança em produção)
- ou uma única chave global para todos os sites

*Para simplificar a configuração, esta documentação assume uma chave global.*

**1. Adicionar as chaves no manifesto**

No SAP Commerce Cloud, as chaves devem ser adicionadas no manifest.json:

    recaptcha.publickey
    recaptcha.privatekey

Essas correspondem às chaves pública e privada geradas no Google.

*No Spartacus, as chaves não precisam ser configuradas manualmente, pois a chave pública pode ser fornecida pelo Commerce através do endpoint de configuração do BaseStore.*

**2. Habilitar o CAPTCHA via BaseStore**

Para ativar a validação de CAPTCHA no Commerce (e permitir que o Spartacus receba a chave pública), é necessário habilitar a flag captchaCheckEnabled no BaseStore.

    INSERT_UPDATE BaseStore; uid[unique=true]; captchaCheckEnabled
                           ; $storeUid       ; true

Uma vez habilitado, o Commerce retornará a chave pública do CAPTCHA na requisição padrão de configuração do BaseStore.

**3. Permitir o header do token CAPTCHA no CORS**

O Spartacus envia o token do CAPTCHA através de um header de requisição. Para permitir esse header nas chamadas OCC, é necessário adicioná-lo aos headers permitidos na configuração de CORS:

sap-commerce-cloud-captcha-token

    INSERT_UPDATE CorsConfigurationProperty; context[unique = true]; key[unique = true]; value
                                           ; commercewebservices   ; allowedHeaders    ; origin content-type accept authorization cache-control if-none-match x-anonymous-consents x-profile-tag-debug x-consent-reference occ-personalization-id occ-personalization-time Sap-Commerce-Cloud-User-Id sap-commerce-cloud-captcha-token

**4. Habilitar a validação de CAPTCHA por endpoint OCC**

Por padrão, habilitar o CAPTCHA não faz com que a validação seja aplicada automaticamente em todas as requisições. Para aplicar a validação de CAPTCHA em um endpoint OCC específico, é necessário adicionar a anotação:

    @CaptchaAware

Essa anotação deve ser colocada acima do método do controller que manipula o endpoint. Isso funciona tanto para:

- endpoints recém-criados
- quanto para endpoints OCC padrão que foram sobrescritos

**Como funciona a validação de CAPTCHA (por trás dos panos)**

Após adicionar a anotação @CaptchaAware em um método de controller OCC, a requisição passa a seguir o fluxo padrão de validação de CAPTCHA do SAP Commerce.

Em tempo de execução, a requisição é interceptada pelo CaptchaValidationInterceptor, que exige que a requisição contenha um token CAPTCHA válido gerado no frontend.

O interceptor então delega a validação para o CaptchaValidationService, que é responsável por validar o token recebido (gerado pelo Google reCAPTCHA no frontend) em relação à chave privada configurada.

Se o token estiver ausente ou for inválido, a requisição é bloqueada e uma resposta de erro é retornada.

**TLDR**

- CAPTCHA é um mecanismo de segurança para prevenir abusos automatizados em aplicações web.
- O fluxo de implementação no SAP Commerce Cloud envolve:
    - Gerar um par de chaves pública/privada no Google reCAPTCHA
    - Configurar ambas as chaves no manifest.json do Commerce (recaptcha.publickey e recaptcha.privatekey)
    - Habilitar o CAPTCHA no BaseStore usando captchaCheckEnabled = true (para que o Spartacus possa receber automaticamente a chave pública)
    - Permitir o header de requisição sap-commerce-cloud-captcha-token na configuração de CORS
    - Aplicar a validação de CAPTCHA apenas nos endpoints necessários adicionando @CaptchaAware
- Por trás dos panos, os métodos de controller anotados são interceptados por CaptchaValidationInterceptor, que exige um token CAPTCHA válido do frontend e o valida via CaptchaValidationService. Se o token estiver ausente ou inválido, a requisição é bloqueada.

**(English)**

**What is it?**

A CAPTCHA is a security mechanism designed to protect web applications from automated abuse (GOOGLE, 2026).
In practice, it helps ensure that a request is being performed by a real user, preventing scenarios such as:

- brute-force login attempts
- spam on forms (e.g., “Contact Us”)
- abusive password reset requests
- automated account creation or enumeration

Google reCAPTCHA is one of the most common CAPTCHA solutions and can work either through:

- v2: interactive challenge (“I’m not a robot”)
- v3: risk score (invisible to the user)

**Getting Started**

To implement CAPTCHA, you must first generate a public/private key pair, created through Google reCAPTCHA administration: https://developers.google.com/recaptcha. When creating the keys, you will need to configure:

- Label: a friendly name to identify where the CAPTCHA is used
- Type:
    - v2 → challenge-based
    - v3 → score-based
- Domains: you may restrict which domains can use the keys
- Project name in Google Developers Console

*It is recommended to have a separate key for local development, without domain restrictions.*

After creating the key pair, Google provides an admin dashboard where you can monitor:

- usage volume
- successful vs failed validations
- suspicious traffic patterns and possible attacks

*The free usage limit is 10,000 validations per month. Above this limit, a paid plan is required.*

**SAP Commerce Configuration**

Once you have the keys, you must decide how you will manage them:

- one key per site (recommended for production governance)
- or a single global key for all sites

*To simplify the setup, this documentation assumes one global key.*

**1. Add the keys to the manifest**

In SAP Commerce Cloud, the keys must be added to the manifest.json:

    recaptcha.publickey
    recaptcha.privatekey

These correspond to the public and private keys generated in Google. 

*In Spartacus, the keys do not need to be configured manually, because the public key can be provided by Commerce through the BaseStore configuration endpoint.*

**2. Enable CAPTCHA via BaseStore**

To activate CAPTCHA validation in Commerce (and allow Spartacus to receive the public key), you must enable the captchaCheckEnabled flag on the BaseStore.

    INSERT_UPDATE BaseStore; uid[unique=true]; captchaCheckEnabled
                           ; $storeUid       ; true

Once enabled, Commerce will return the CAPTCHA public key in the standard BaseStore configuration request.

**3. Allow the CAPTCHA token header in CORS**

Spartacus sends the CAPTCHA token through a request header. To allow this header in OCC calls, you must add it to the allowed headers in CORS configuration:

sap-commerce-cloud-captcha-token

    INSERT_UPDATE CorsConfigurationProperty; context[unique = true]; key[unique = true]; value
                                           ; commercewebservices   ; allowedHeaders    ; origin content-type accept authorization cache-control if-none-match x-anonymous-consents x-profile-tag-debug x-consent-reference occ-personalization-id occ-personalization-time Sap-Commerce-Cloud-User-Id sap-commerce-cloud-captcha-token

**4. Enable CAPTCHA validation per OCC endpoint**

By default, enabling CAPTCHA does not automatically enforce validation on every request. To enforce CAPTCHA validation on a specific OCC endpoint, you must add the annotation:

    @CaptchaAware


This annotation should be placed above the controller method handling the endpoint. This works both for:

- newly created endpoints
- overridden standard OCC endpoints

**How CAPTCHA Validation Works (Behind the Scenes)**

After adding the @CaptchaAware annotation to an OCC controller method, the request starts going through the standard SAP Commerce CAPTCHA validation flow.

At runtime, the request is intercepted by the CaptchaValidationInterceptor, which enforces that the request contains a valid CAPTCHA token generated on the frontend.

The interceptor then delegates the validation to the CaptchaValidationService, which is responsible for validating the received token (generated by Google reCAPTCHA on the frontend) against the configured private key.

If the token is missing or invalid, the request is blocked and an error response is returned.

**TLDR**

- CAPTCHA is a security mechanism to prevent automated abuse of web applications.
- The implementation flow in SAP Commerce Cloud involves:
    - Generate a public/private key pair in Google reCAPTCHA
    - Configure both keys in Commerce manifest.json (recaptcha.publickey and recaptcha.privatekey)
    - Enable CAPTCHA in the BaseStore using captchaCheckEnabled = true (so Spartacus can automatically receive the public key)
    - Allow the request header sap-commerce-cloud-captcha-token in CORS configuration
    - Enforce CAPTCHA validation only on required endpoints by adding @CaptchaAware
- Under the hood, annotated controller methods are intercepted by CaptchaValidationInterceptor, which requires a valid frontend CAPTCHA token and validates it via CaptchaValidationService. If the token is missing or invalid, the request is blocked.


**SOURCE**
---

GOOGLE. What is reCAPTCHA?. Available at: <https://developers.google.com/recaptcha>. Accessed on: Feb. 17, 2026.

SAP. CAPTCHA OCC APIs. Available at: <https://help.sap.com/docs/SAP_COMMERCE_CLOUD_PUBLIC_CLOUD/e1391e5265574bfbb56ca4c0573ba1dc/8a42681cd28549b6a81d6faffc7d6b92.html>. Accessed on: Feb. 17, 2026.