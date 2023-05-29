# Mise en oeuvre d'un micro-service 


### Dépendances :
- Spring Web
- Spring Data JPA
- Spring GraphQL
- H2 Database
- Lombok

## Entities :
- BankAccount
```java
@Entity @NoArgsConstructor @AllArgsConstructor @Builder
@Getter @Setter
public class BankAccount {
    @Id
    private String id;
    @Temporal(javax.persistence.TemporalType.DATE)
    private Date creationAt;
    private Double balance;
    private String currency;
    @Enumerated(EnumType.STRING)
    private AccountType type;

}
```

## Repositories:


- BankAccountRepository
```java
@RepositoryRestResource
public interface BankAccountRepository extends JpaRepository<BankAccount, String>{
    @RestResource(path = "/byType")
    List<BankAccount> findByType(@Param("t") AccountType type);
    List<BankAccount> findByCurrency(String currency);
}
```

## DTOs

- BankAccountRequestDTO
```java
@Data @NoArgsConstructor @AllArgsConstructor @Builder
public class BankAccountRequestDTO {
    private Double balance;
    private String currency;
    private AccountType type;
}
```
- BankAccountResponseDTO
```java
@Data @NoArgsConstructor @AllArgsConstructor @Builder
public class BankAccountResponseDTO {
    private String id;
    private Date creationAt;
    private Double balance;
    private String currency;
    private AccountType type;
}
```

## Mappers
- AccountMapper
```java
@Component
public class AccountMapper {
    public BankAccountResponseDTO fromBankAccount(BankAccount bankAccount) {
        BankAccountResponseDTO bankAccountResponseDTO = new BankAccountResponseDTO();
        BeanUtils.copyProperties(bankAccount, bankAccountResponseDTO);
        return bankAccountResponseDTO;
    }

}

```

## Création du service DAO
On crée le package `com.example.bankaccount.service` et on y ajoute l'interface suivante :
- AccountService
```java
public interface AccountService {
    BankAccountResponseDTO addAccount(BankAccountRequestDTO bankAccountDTO);
}
```
On crée ensuite l'implémentation de cette interface :
- AccountServiceImpl
```java
@Service
@Transactional
public class AccountServiceImpl implements AccountService {
    @Autowired
    private BankAccountRepository bankAccountRepository;
    @Autowired
    private AccountMapper accountMapper;
    @Override
    public BankAccountResponseDTO addAccount(BankAccountRequestDTO bankAccountDTO) {
        BankAccount bankAccount = BankAccount.builder()
                .id(UUID.randomUUID().toString())
                .creationAt(new java.util.Date())
                .balance(bankAccountDTO.getBalance())
                .type(bankAccountDTO.getType())
                .currency(bankAccountDTO.getCurrency())
                .build();
        BankAccount savedBankAccount=bankAccountRepository.save(bankAccount);
        return accountMapper.fromBankAccount(savedBankAccount);
    }
}

```

- AccountRestController
```java
@RestController
@RequestMapping("/api")
public class AccountRestController {
    private BankAccountRepository bankAccountRepository;
    private AccountService accountService;
    private AccountMapper accountMapper;

    public AccountRestController(BankAccountRepository bankAccountRepository, AccountService accountService, AccountMapper accountMapper) {
        this.bankAccountRepository = bankAccountRepository;
        this.accountService = accountService;
        this.accountMapper = accountMapper;
    }

    @GetMapping("/bankAccounts")
    public List<BankAccount> bankAccounts(){
        return bankAccountRepository.findAll();
    }
    @GetMapping("/bankAccounts/{id}")
    public BankAccount bankAccounts(String id){
        return bankAccountRepository.findById(id).orElseThrow(()->new RuntimeException(String.format("Account %s not found",id)));
    }

    /*En utilisant la projection*/
    @PostMapping("/bankAccounts")
    public BankAccountResponseDTO save(@RequestBody BankAccountRequestDTO requestDTO){
        return accountService.addAccount(requestDTO);
    }

    @PutMapping("/bankAccounts/{id}")
    public BankAccount update(String id, @RequestBody BankAccount bankAccount){
        BankAccount account = bankAccountRepository.findById(id).orElseThrow();
        if (bankAccount.getBalance()!=null) account.setBalance(bankAccount.getBalance());
        if (bankAccount.getCurrency()!=null) account.setCurrency(bankAccount.getCurrency());
        if (bankAccount.getType()!=null) account.setType(bankAccount.getType());
        if (bankAccount.getCreationAt()!=null) account.setCreationAt(new Date());
        return bankAccountRepository.save(account);
    }

}
```


- schema.graphqls
```graphql
type Query {
    accountsList: [BankAccount]
}

type BankAccount {
    id: String,
    createdAt: Float,
    balance: Float,
    currency: String,
    type: String
}
```

- AccountGraphQLController
```java
@Controller
public class AccountGraphQLController {
    @Autowired
    private BankAccountRepository bankAccountRepository;
    @QueryMapping
    public List<BankAccount> accountsList(){
        return bankAccountRepository.findAll();
    }

}
```


#### schema.graphqls :
```graphql
type Mutation {
    addAccount(bankAccount: BankAccountDTO): BankAccount,
    updateAccount(id: String, bankAccount: BankAccountDTO): BankAccount,
    deleteAccount(id: String): Boolean
}

input BankAccountDTO {
    balance: Float,
    currency: String,
    type: String
---
