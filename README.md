Claro! Aqui está a implementação ajustada e formatada adequadamente para ser copiada e colada diretamente no seu repositório GitHub.

### Estrutura de Diretórios

- src/
  - entities/
    - customer.entity.ts
    - proposal.entity.ts
    - user.entity.ts
  - controllers/
    - proposal.controller.ts
  - services/
    - proposal.service.ts
  - middlewares/
    - get-user.middleware.ts
  - proposal.module.ts

### 1. Definindo as Entidades

#### `src/entities/customer.entity.ts`
```typescript
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { Proposal } from './proposal.entity';

@Entity()
export class Customer {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @Column({ default: 0 })
    balance: number;

    @OneToMany(() => Proposal, proposal => proposal.customer)
    proposals: Proposal[];
}
```

#### `src/entities/proposal.entity.ts`
```typescript
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne } from 'typeorm';
import { Customer } from './customer.entity';
import { User } from './user.entity';

export enum ProposalStatus {
    PENDING = 'PENDING',
    REFUSED = 'REFUSED',
    ERROR = 'ERROR',
    SUCCESSFUL = 'SUCCESSFUL',
}

@Entity()
export class Proposal {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    customerId: number;

    @ManyToOne(() => Customer, customer => customer.proposals)
    customer: Customer;

    @Column()
    userId: number;

    @ManyToOne(() => User, user => user.proposals)
    user: User;

    @Column({ type: 'enum', enum: ProposalStatus, default: ProposalStatus.PENDING })
    status: ProposalStatus;

    @Column('decimal')
    value: number;

    @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
    createdAt: Date;
}
```

#### `src/entities/user.entity.ts`
```typescript
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { Proposal } from './proposal.entity';

@Entity()
export class User {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    username: string;

    @Column({ default: 0 })
    totalProfit: number;

    @OneToMany(() => Proposal, proposal => proposal.user)
    proposals: Proposal[];
}
```

### 2. Criando o Controlador

#### `src/controllers/proposal.controller.ts`
```typescript
import { Controller, Get, Post, Param, UseGuards, Req } from '@nestjs/common';
import { ProposalService } from '../services/proposal.service';
import { GetUser } from '../middlewares/get-user.middleware'; // Ajuste o caminho se necessário
import { Request } from 'express';

@Controller('proposals')
export class ProposalController {
    constructor(private readonly proposalService: ProposalService) {}

    @Get(':id')
    @UseGuards(GetUser)
    async getProposal(@Param('id') id: number, @Req() req: Request) {
        return this.proposalService.getProposal(id, req.user.id);
    }

    @Get()
    @UseGuards(GetUser)
    async getPendingProposals(@Req() req: Request) {
        return this.proposalService.getPendingProposals(req.user.id);
    }

    @Get('refused')
    @UseGuards(GetUser)
    async getRefusedProposals(@Req() req: Request) {
        return this.proposalService.getRefusedProposals(req.user.id);
    }

    @Post(':proposal_id/approve')
    @UseGuards(GetUser)
    async approveProposal(@Param('proposal_id') proposalId: number, @Req() req: Request) {
        return this.proposalService.approveProposal(proposalId, req.user.id);
    }
}
```

### 3. Implementando o Serviço

#### `src/services/proposal.service.ts`
```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Proposal } from '../entities/proposal.entity';
import { User } from '../entities/user.entity';
import { Customer } from '../entities/customer.entity';

@Injectable()
export class ProposalService {
    constructor(
        @InjectRepository(Proposal)
        private proposalRepository: Repository<Proposal>,
        @InjectRepository(User)
        private userRepository: Repository<User>,
        @InjectRepository(Customer)
        private customerRepository: Repository<Customer>,
    ) {}

    async getProposal(id: number, userId: number) {
        const proposal = await this.proposalRepository.findOne({ where: { id, userId } });
        if (!proposal) throw new Error('Proposal not found or does not belong to the user.');
        return proposal;
    }

    async getPendingProposals(userId: number) {
        return this.proposalRepository.find({ where: { userId, status: 'PENDING' } });
    }

    async getRefusedProposals(userId: number) {
        return this.proposalRepository.find({ where: { userId, status: 'REFUSED' } });
    }

    async approveProposal(proposalId: number, userId: number) {
        const proposal = await this.proposalRepository.findOne({ where: { id: proposalId, userId, status: 'PENDING' } });
        if (!proposal) throw new Error('Proposal not found or does not belong to the user.');

        proposal.status = 'SUCCESSFUL';
        await this.proposalRepository.save(proposal);

        // Atualizar o lucro do usuário
        const user = await this.userRepository.findOne({ where: { id: userId } });
        user.totalProfit += proposal.value;
        await this.userRepository.save(user);

        return proposal;
    }
}
```

### 4. Integrando com o Módulo

#### `src/proposal.module.ts`
```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ProposalController } from './controllers/proposal.controller';
import { ProposalService } from './services/proposal.service';
import { Proposal } from './entities/proposal.entity';
import { User } from './entities/user.entity';
import { Customer } from './entities/customer.entity';

@Module({
    imports: [TypeOrmModule.forFeature([Proposal, User, Customer])],
    controllers: [ProposalController],
    providers: [ProposalService],
})
export class ProposalModule {}
```

### 5. Configurando o Middleware

#### `src/middlewares/get-user.middleware.ts`
```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { User } from '../entities/user.entity';
import { UserService } from '../services/user.service'; // Supondo que você tenha esse serviço

@Injectable()
export class GetUser implements NestMiddleware {
  constructor(private userService: UserService) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const userId = req.headers['user_id']; // Ajuste conforme necessário

    if (!userId) {
      return res.status(401).send('Unauthorized');
    }

    const user: User = await this.userService.findById(userId);
    if (!user) {
      return res.status(404).send('User not found');
    }

    req.user = user; // Adiciona o usuário à requisição
    next();
  }
}
```

### Finalizando

Agora, você pode copiar e colar essas seções diretamente em seus arquivos do repositório GitHub. Certifique-se de seguir as melhores práticas ao fazer commits e também de documentar seu projeto adequadamente.

Se precisar de mais ajuda em outros aspectos do projeto, sinta-se à vontade para perguntar!
