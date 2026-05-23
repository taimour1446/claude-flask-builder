# Signature Patterns

## 1. BaseResponse envelope (LOCKED for client compat)
Every JSON response: `{status, data, error, authError, force_update, message, request_id}`
- `respond(data, message?)` → error=0, status=200
- `respondError(message, authError=0, force_update=0)` → error=1, status=200
- `request_id` populated by RequestInterceptor's X-Request-ID
- `trace` field ONLY in dev — never in production response

## 2. Controller skeleton (R23)
```python
@blueprint.route('/path', methods=['POST'])
@token_required  # if auth required
def handler(current_user) -> BaseResponse:
    try:
        data = XService.method(request, current_user)
        return BaseResponse.respond(data)
    except CustomValidationException as e:
        return BaseResponse.respondError(message=str(e))
    except Exception as e:
        logger.exception('handler_failed')
        return BaseResponse.respondError(message='Internal error')
```
Public routes omit `@token_required` AND `current_user` param.
Raw returns (files/exports/health) skip BaseResponse — sanctioned exception.

## 3. Service skeleton (R24)
```python
class XService:
    @staticmethod
    def method(request, current_user=None):
        data = Validation.validate(request, CreateXValidation())
        Auth.assert_role(current_user, [Role.Admin])  # if role-gated
        with db.session.begin_nested():  # SAVEPOINT for multi-model
            obj = X.create(data)
            Y.create(...)
        return XSchema().dump(obj)
```
Service composes multi-model writes in ONE transaction (R25).
Service NEVER touches request beyond passing to Validation.validate.

## 4. Validation schema (R26, R28)
```python
class CreateXValidation(Schema):
    name = fields.String(required=True, error_messages={'required': 'Name is required'})

    class Meta:
        unknown = EXCLUDE  # not INCLUDE — R62

    @validates_schema
    def custom(self, data, **kwargs):
        if X.exists(data['name']):
            raise CustomValidationException(Constants.NameTaken)
```

## 5. DTO schema (R27)
```python
class XSchema(Schema):
    id = fields.Int()
    name = fields.Str()

    @post_dump
    def derive(self, data, **kwargs):
        # NO DB queries here — pre-join in the service
        if data == {}:
            return None
        data['full_name'] = f"{data.get('first_name', '')} {data.get('last_name', '')}".strip()
        return data
```

## 6. Model skeleton (R25, R43, R44)
```python
class X(db.Model):
    __tablename__ = 'xs'  # plural
    id = Column(Integer, primary_key=True)
    name = Column(Text, nullable=False)
    amount = Column(Numeric(10, 2), nullable=False, default=0)  # R45
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
    deleted_at = Column(DateTime(timezone=True), nullable=True)  # R47

    def to_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}

    @staticmethod
    def create(data):
        obj = X(name=data['name'], amount=data.get('amount', 0))  # explicit fields, no __dict__.update
        db.session.add(obj)
        db.session.commit()
        return obj

    @staticmethod
    def get_by_id(id):
        return db.session.query(X).filter_by(id=id, deleted_at=None).first()
```

## 7. JWT auth (R60)
- Auth.generate_token(user) returns access + refresh tokens
- Access TTL 30 min, refresh 30 days, rotation on use
- token_required decorator validates exp, loads user, injects current_user

## 8. RequestInterceptor (R67, R91)
- before_request: generate request_id, log redacted body
- after_request: attach X-Request-ID header, update log status
- Redact: password, ptoken, otp, card_number, cvc, stripe_secret

## 9. Service-direct cron jobs (P6-8)
NO self-callback `requests.post(BASE_URL + ...)` pattern.
Cron jobs in `scheduler_jobs/` call services directly:
```python
@scheduler.task('cron', id='purge_pending_orders', hour=0, timezone='UTC', max_instances=1)
def purge_pending_orders():
    with scheduler.app.app_context():
        with distributed_lock('purge_pending_orders'):
            OrderService.purge_pending(...)
```
