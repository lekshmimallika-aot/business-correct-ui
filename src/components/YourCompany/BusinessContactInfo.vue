<template>
  <div>
    <v-layout row v-if="!isEditing">
      <v-flex md3>
        <label><strong>Registered Office Information</strong></label>
      </v-flex>
      <v-flex md4>
        <div>
          <label><strong>Email Address</strong></label>
        </div>
        <div id="lbl-email">{{ !!contact.email ? contact.email : "(Not entered)" }}</div>
        <div v-if="hasEmailAddressChange">
          <v-chip x-small label color="#1669BB" text-color="white" id="email-corrected-lbl">
            Corrected
          </v-chip>
        </div>
      </v-flex>
      <v-flex md3>
        <div>
          <label><strong>Phone Number</strong></label>
        </div>
        <div id="lbl-phone" v-if="!!contact.phone">
          {{ contact.phone }}
          <span v-if="!!contact.extension">Ext: {{ contact.extension }}</span>
        </div>
        <div id="lbl-phone" v-else>(Not entered)</div>
        <div v-if="hasPhoneNumberChange">
          <v-chip x-small label color="#1669BB" text-color="white" id="phone-corrected-lbl">
            Corrected
          </v-chip>
        </div>
      </v-flex>
      <v-flex md2 class="pl-9">
        <v-btn
          v-if="!hasBusinessContactInfoChange"
          text
          color="primary"
          id="btn-correct-contact-info"
          @click="isEditing = true">
          <v-icon small>mdi-pencil</v-icon>
          <span>Correct</span>
        </v-btn>
        <v-btn
          v-if="hasBusinessContactInfoChange"
          text
          color="primary"
          id="btn-undo-contact-info"
          @click="resetContactInfo">
          <v-icon small>mdi-undo</v-icon>
          <span>Undo</span>
        </v-btn>
      </v-flex>
    </v-layout>
    <div v-else>
      <v-layout row>
        <v-flex md3>
          <label><strong>Registered Office Information</strong></label>
        </v-flex>
      </v-layout>
      <v-form v-model="formValid" ref="form" name="business-contact-form" class="business-contact-form">
        <v-layout row>
          <v-flex md2>
            <label><strong>Email Address</strong></label>
          </v-flex>
          <v-flex md10>
            <v-text-field
              filled
              label="Email Address"
              req
              persistent-hint
              :rules="emailRules"
              v-model="contact.email"
              id="txt-email">
            </v-text-field>
          </v-flex>
        </v-layout>
        <v-layout row>
          <v-flex md2>
            <label><strong>Confirm Email</strong></label>
          </v-flex>
          <v-flex md10>
            <v-text-field
              filled
              label="Confirm Email Address"
              req
              persistent-hint
              :error-messages="emailMustMatch()"
              v-model="contact.confirmEmail"
              id="txt-confirm-email">
            </v-text-field>
          </v-flex>
        </v-layout>
        <v-layout row>
          <v-flex md2>
            <label><strong>Phone Number</strong></label>
          </v-flex>
          <v-flex md6 class="pr-3">
            <v-text-field
              filled
              label="Phone Number"
              persistent-hint
              hint="Example: (555) 555-5555"
              type="tel"
              v-mask="['(###) ###-####']"
              v-model="contact.phone"
              :rules="phoneRules"
              id="txt-phone">
            </v-text-field>
          </v-flex>
          <v-flex>
            <v-text-field
              filled
              label="Extension"
              persistent-hint
              v-mask="'#####'"
              v-model="contact.extension"
              :disabled="!contact.phone"
              id="txt-phone-extension">
            </v-text-field>
          </v-flex>
        </v-layout>
        <v-layout row>
          <v-flex md12>
            <div class="action-btns">
              <v-btn id="done-btn" large color="primary" @click="updateContactInfo" :disabled="!formValid">
                <span>Done</span>
              </v-btn>
              <v-btn id="cancel-btn" large outlined color="primary" @click="resetContactInfo">
                <span>Cancel</span>
              </v-btn>
            </div>
          </v-flex>
        </v-layout>
      </v-form>
    </div>
  </div>
</template>

<script lang="ts">
// Libraries
import { Component, Vue, Prop, Watch, Emit, Mixins } from 'vue-property-decorator'
import { mask } from 'vue-the-mask'

// Interfaces
import { ActionBindingIF, BusinessContactIF, IncorporationFilingIF } from '@/interfaces'

// Mixins
import { CommonMixin } from '@/mixins'
import { Action, Getter } from 'vuex-class'

@Component({
  directives: { mask }
})
export default class BusinessContactInfo extends Mixins(CommonMixin) {
  @Prop()
  private businessContact!: BusinessContactIF

  @Prop()
  private originalBusinessContact!: BusinessContactIF

  // Properties
  private isEditing: boolean = false
  private contact: BusinessContactIF
  private formValid: boolean = false

  // Rules
  private emailRules = [
    (v: string) => !!v || 'Email address is required',
    (v: string) => {
      const pattern = /^(([^<>()[\]\\.,;:\s@"]+(\.[^<>()[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/
      return pattern.test(v) || 'Valid email is required'
    }
  ]

  private phoneRules = [
    (v: any) => !v || (v.length === 0 || v.length === 14) || 'Phone number is invalid'
  ]

  private emailMustMatch (): string {
    return (this.contact.email === this.contact.confirmEmail) ? '' : 'Email addresses must match'
  }

  private get hasBusinessContactInfoChange () {
    return !this.isSame(this.businessContact, this.getOriginalBusinessContact())
  }

  // Watchers
  @Watch('businessContact', { deep: true, immediate: true })
  private onContactPropValueChanged (): void {
    this.contact = { ...this.businessContact }
    this.emitHaveChanges(this.hasBusinessContactInfoChange)
  }

  private updateContactInfo (): void {
    this.emitContactInfo(this.contact)
    this.isEditing = false
    this.emitHaveChanges(this.hasBusinessContactInfoChange)
  }

  private resetContactInfo (): void {
    this.contact = this.getOriginalBusinessContact()
    this.emitContactInfo(this.contact)
    this.emitHaveChanges(false)
    this.isEditing = false
  }

  private getOriginalBusinessContact (): BusinessContactIF {
    return {
      email: this.originalBusinessContact.email,
      confirmEmail: this.originalBusinessContact.email,
      phone: this.originalBusinessContact.phone,
      extension: this.originalBusinessContact.extension
    }
  }

  private get hasPhoneNumberChange (): boolean {
    return this.businessContact.phone !== this.originalBusinessContact.phone ||
    this.businessContact.extension !== this.originalBusinessContact.extension
  }

  private get hasEmailAddressChange (): boolean {
    return this.businessContact.email !== this.originalBusinessContact.email
  }

  @Emit('contactInfoChange')
  private emitContactInfo (contactInfo: BusinessContactIF): void { }

  @Emit('haveChanges')
  private emitHaveChanges (haveChanges: boolean): void { }
}
</script>

<style lang="scss" scoped>
  [class^="col"] {
    padding-top: 0;
    padding-bottom: 0;
  }

  .business-contact-form {
    margin-top: 1rem;
    padding: 1.25rem;
    padding-left: 0;
    margin-right: 1rem;
  }

  .action-btns {
    display: flex;
    justify-content: flex-end;
    padding-bottom: 1rem;
    padding-right: 0.5rem;

    .v-btn + .v-btn {
      margin-left: 0.5rem;
    }

    .v-btn {
      min-width: 6.5rem;
    }

    .v-btn[disabled] {
      color: white !important;
      background-color: #1669bb !important;
      opacity: 0.2;
    }
  }
</style>
