# Log in in Swagger / Nelmio API Doc

You only have to configure nelmio API doc at `config/packages/nelmio_api_doc.yml` to add endpoints to our documentation.
Let's add the login and refresh token routes : 
```yaml
nelmio_api_doc:
    documentation:
        info:
            title: Our API
            description: Our API
            version: 1.0.0
        securityDefinitions:
            Bearer:
                type: apiKey
                description: 'Value: Bearer {jwt}'
                name: Authorization
                in: header
        security:
            - Bearer: []
        paths:
            /api/login_check:
                post:
                    tags:
                        - Login
                    description: Login into the API.
                    produces:
                        - application/json
                    parameters:
                        - name: user
                          description: User to login
                          in: body
                          required: true
                          schema:
                              type: object
                              properties:
                                  username:
                                      type: string
                                  password:
                                      type: string
                    responses:
                        '200':
                            description: Login successful
                            schema:
                                type: object
                                properties:
                                    token:
                                        type: string
                                    refresh_token:
                                        type: string
            /api/token/refresh:
                post:
                    tags:
                        - Login
                    description: Refresh user's token
                    produces:
                        - application/json
                    parameters:
                        - name: refresh_token_data
                          description: refresh_token
                          in: body
                          required: true
                          schema:
                              type: object
                              properties:
                                  refresh_token:
                                      type: string
                    responses:
                        '200':
                            description: Token successfully refreshed
                            schema:
                                type: object
                                properties:
                                    token:
                                        type: string
                                    refresh_token:
                                        type: string
```
